# Akka Notes

Akka is an open source toolkit and runtime that makes it easier to build concurrent, parallel and distributed applications on the JVM.

Akka provides:
-   Multi-threaded behavior without the use of low-level concurrency constructs like atomics or locks
-   Transparent remote communication between systems and their components
-   A clustered, high-availability architecture that elastically scales on demand

## Multithreading in Scala
On the JVM a new thread is created with a Thread constructor, which receives a Runnable object in which the `run` method does something.

```Scala
val aThread = new Thread(new Runnable {
	override def run(): Unit = println("I'm running in parallel")
})

// more concise syntax
val aThread = new Thread(() => println("I'm running in parallel"))

aThread.start() // start a thread
aThread.join() // wait for a thread
```

Threads are great for making use of the power of multicore processors.

The problem with threads is that they're unpredictable. If you run two threads it's completely unpredictable which order they'll run in.

Also, different runs will produce different results.

The way you normally make threads safe is by adding ``synchronized`` blocks around the critical instructions. In a synchronized expression, two threads can't evaluate it at the same time.

You could also add `@volatile` to the private member which would make reads atomic, preventing two threads from reading the value at the same time. This onyl works for primitive types such as Int.

```Scala
class BankAccount(@volatile private var amount: Int) {  
	override def toString: String = "" + amount  
  
	def withdraw(money: Int) = this.amount -= money  
  
	def safeWithdraw(money: Int) = this.synchronized {  
	  this.amount -= money  
	}  
}
```

Inter-thread communication on the JVM is done via the wait and notify mechanism.

#### Scala Futures
A Future represents a value which may or may not _currently_ be available, but will be available at some point, or an exception if that value could not be made available.

For long running tasks, their value may or may not be currently available but will be available at some point in the future.

From a functional programming perspective, Future is a monadic construct, meaning it has functional primitives: `map`, `flatMap` and `filter` as well as for comprehensions.

```Scala
// futures
import scala.concurrent.ExecutionContext.Implicits.global

val future = Future {
	// long computation - on a different thread
	42
}

// callbacks
future.onComplete {
	case Success(42) => println("I found the meaning of life")
	case Failure(_) => println("something happened with the meaning of life!")
}

val aProcessedFuture = future.map(_ + 1) // Future with 43

val aFlatFuture = future.flatMap { value =>
	Future(value + 2)
} // Future with 44

val filteredFuture = future.filter(_ % 2 == 0) // NoSuchElementException

// for comprehensions

val aNonsenseFuture = for {
	meaningOfLife <- future
	filteredMeaning <- filteredFuture
} yield meaningOfLife + filteredMeaning
```

### Thread Model Limitations
OOP encapsulation is arguably only valid in the single-threaded model because of race conditions. Locks solve one problem but introduce others, such as deadlocks and livelocks.

You would need a data structure that is fully encapsulated and with no locks.

Delegating something to an already running thread is a pain. What if you need to send other signals? What if there are multiple background tasks? How do you identify who gave the signal? What if the background thread crashes?

You would need a data structure that can safely receive messages, can identify the sender, is easily identifiable, and can guard against errors.

Tracing and dealing with errors in a multithreaded environment is a pain, even in small systems.

## Akka Actors

With traditional objects, you model your code around instances of classes.

With objects:
- you store their state as data
- you call their methods

With actors:
- you store their state as data
- you send messages to them, asynchronously

Actors are objects you can't access directly, but can only send messages to.

Working with actors is like asking someone for information and waiting for their response.

Every interaction happens via sending and receiving messages.

These messages are asynchronous by nature:
- it takes time for a message to travel
- receiving and responding may not happen at the same time
- sending and receiving might not even happen in the same context

### Actors, Messages and Behaviours
Every Akka application starts with an ActorSystem.

An ActorSystem is a heavyweight data structure that controls a number of threads under the hood and allocates them to running actors.

You should have one ActorSystem per application unless you have a good reason to create more. The ActorSystem's name must contain only alphanumeric characters and non-leading hyphens or underscores. Actors can be located by their actor system.

```Scala
val actorSystem = ActorSystem("firstActorSystem")

println(actorSystem.name) // firstActorSystem
```

- Actors are uniqualy identified
- Messages are asynchronously
- Each actor has a unique way of processing the message
- Actors are (really) encapsulated

### Creating an Actor
You create an actor by creating a class that extends the ``Actor`` trait.

An actor needs a `receive` method. It takes no arguments and its return type is ` PartialFunction[Any, Unit]` also aliased by the type `Receive`.

You communicate with an actor by instantiating it (really an ActorRef) and then sending it a message.

You can't instantiate an actor by calling new but by invoking the ActorSystem. You pass in a Props object (typed with the actor type you want to instantiate) and the name for the actor.

The name restrictions for actors are the same as for the ActorSystem.

Instantiating an actor produces an ActorRef, the data structure that Akka exposes to you since you can't call the actual actor instance itself. You can only communicatew with an actor via an ActorRef.

The method to invoke an ActorRef is `!`, also known as `tell` (Scala is very permissive about method naming).

```Scala
class WordCountActor extends Actor {
	// internal data
	var totalWords = 0

	// behavior
	def receive: Receive = {
		case message: String =>
			println(s"[word counter] I have received: $message")
			totalWords += message.split(" ").length
		case msg => println(s"[word counter] I cannot understand ${msg.toString}")
	}
}

// instantiating
val wordCounter = actorSystem.actorOf(Props[WordCountActor], "wordCounter")

val anotherWordCounter = actorSystem.actorOf(Props[WordCountActor], "anotherWordCounter")

// communicating - these are completely asynchronous
wordCounter ! "I am learning Akka and it's pretty cool!" // "tell"
anotherWordCounter ! "A different message"
```

### Actors with Constructor Arguments
You can instantiate an actor with constructor arguments using Props with an argument.

The best practice is to declare a companion object and define a method that returns a Props object. You don't create actor instances yourself. Instead the factory method creates Props with actor instances for you.

```Scala
// best practice for creating actors with constructor arguments
object Person {  
  def props(name: String) = Props(new Person(name))  
}  
  
class Person(name: String) extends Actor {  
	override def receive: Receive = {  
	  case "hi" => println(s"Hi, my name is $name")  
	  case _ =>  
	}  
}  
  
val person = actorSystem.actorOf(Person.props("Bob"))  
person ! "hi"
```

### Messages
Messages can be of (almost) any type.

You can send any primitive type by default. You can also define your own.

When you invoke the tell method, Akka retrieves the object that will then be invoked on the message type that you sent.

Because an actor uses a partial function you can include any number of cases and the message will be subject to pattern matching.

```Scala
class SimpleActor extends Actor {
	override def receive: Receive = {
		case "Hi!" => sender() ! "Hello, there!" // replying to a message
		case message: String => println(s"[$self] I have received $message")
		case number: Int => println(s"[simple actor] I have received a NUMBER: $number")
		case SpecialMessage(contents) => println(s"[simple actor] I have received something SPECIAL: $contents")
		case SendMessageToYourself(content) =>
			self ! content
		case SayHiTo(ref) => ref ! "Hi!" // alice is being passed as the sender
		case WirelessPhoneMessage(content, ref) => ref forward (content + "s") // keeps the original sender of the WPM
	}
}
```

Messages can be almost any type but:
- they must be immutable
- they must be serializable

You need to enforce this principle since it currently can't be checked for at compile time.

In practice, you'll use case classes and case objects for almost all your message needs.

### Context
Actors have information about their context and about themselves.

Each actor has a member called `context`, a complex data structure with information about the environment the actor runs in.

`context.self` gives you access to the actors own ActorRef (the equivalen tof `this` in the object-oriented world). 

You can use `self` to have an actor send a message to itself.

Actors can reply to messages using `context`. `context.sender()` returns an ActorRef which you can then use to send a message back.

For every actor, at any moment in time `context.sender()` contains the ActorReference of the last actor that sent a message to it. `context.sender()` can also be written as `sender`.

Whenever an actor sends a message to another actor, they pass themselves as the sender.

The tell method receives the message and an implicit sender parameter, `Actor.noSender`, which is null. Since `self` is an implicit value, you usually omit it as a parameter.

```Scala
// under the hood
def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit

final val noSender: ActorRef = null

// inside SinmpleActor
case SayHiTo(ref) => ref ! "Hi!" // equivalent to (ref ! "Hi!")(self)
```

if there is no sender, the reply will go to a fake actor called  `dead letters` because `Actor.noSender` has a default value of null.

Actors can forward messages to one another.

```Scala
case class WirelessPhoneMessage(content: String, ref: ActorRef)  
alice ! WirelessPhoneMessage("Hi", bob) // noSender.

// inside SimpleActor
case WirelessPhoneMessage(content, ref) => ref forward (content + "s") // keeps the original sender of the WPM
```

### Actor Recap
Every actor derives from the `Actor` trait, which has the abstract method `receive`.

`receive` returns a message handler object, which is retrieved by Akka when an actor receives a message.

This handler is invoked when the actor processes a message.

The `Receive` type is an alias of `PartialFunction[Any, Unit]`.

Actors need infrastructure in the form of an ActorSystem.

```Scala
val system = ActorSystem("AnActorSystem")
```

Creating an actor is done via the actor system, not via a new constructor. You need to call the `àctorOf` factory method from the system, passing in a Props object ( a data structure with create/deploy information).

```Scala
val actor = system.actorOf(Props[MyActor], "myActorName")
```

The only way you can communicate with an actor is by sending messages by invoking the tell method, `!`, with the message that you want to send.

Messages can be of any type as long as they are immutable and serializable.

By its design, Akka enforces the actor principles:

- Actors are fully encapsulated. You cannot create actors manually and cannot directly call their methods.
- Actors run in parallel.
- Actors react to messages in a non-blocking and asynchronous manner.

You can communicate with actors using actor references. These can be sent as parts of messages.

Actors are aware of their own reference using `self`.

Actors are aware of the actor reference that last sent them a message using `sender()` and can use this to reply to messages: `sender()` ! "well, hello there".

It's a good practice to put messages in the companion object of the actor that supports them.

```Scala
// the domain of the actor
object Counter {  
	case object Increment  
	case object Decrement  
	case object Print  
}  
  
class Counter extends Actor {  
	// import everything from the companion object 
	import Counter._  
  
  var count = 0  
  
	override def receive: Receive = {  
	  case Increment => count += 1  
		case Decrement => count -= 1  
		case Print => println(s"[counter] My current count is $count")  
	}  
}  

// import from counter domain
import Counter._  
val counter = system.actorOf(Props[Counter], "myCounter")  
  
(1 to 5).foreach(_ => counter ! Increment)  
(1 to 3).foreach(_ => counter ! Decrement)  
counter ! Print
```

### How Actors Work
Actors raise some valid questions:
- Can we assume any ordering of messages?
- Aren't we causing race conditions?
- What does **asynchronous** actually mean for actors?
- How does this work with the JVM?

Akka has a thread pool that it shares with actors.

An actor has both a message handler and a message queue (mailbox). Whenever you send a message, its enqueued in this mailbox.

An actor is a data structure, it's passive and needs a thread to run.

Akka spawns a small number of treads (100s) ehich can handle a large amount of actors (1000000s per GB heap).

Akka schedules actors for execution on these threads.

When you send a message to an actor it's enqueued in the actor's mailbox. This is thread-safe.

To process a message, Akka schedules a thread to run this actor.

Messages are extracted (dequeued) from the mailbox, in order.

For each message, the thread invokes the message handler. As a result, the actor might change its state or send messages to other actors. After that, the message is discarded and the process happens again.

At some point the Akka thread scheduler unschedules the actor, at which point the thread releases control of this actor and moves on to do something else.

This process provides certain guarantees:
- only one thread operates on an actor at any time (actors are effectively single threaded so no locks are needed)
- the thread may never release the actor in the middle of processing messages (processing messages is atomic)

The message delivery environment is inherently unreliable but:
- Akka offers at most once delivery (the actor will never receive duplicates of a message)
- for any sender-receiver pair, the message order is maintained

If Alice sends Bob message A followed by message B:
- Bob will never receive duplicates of A or B
- Bob will **always** receive A before B (possibly with some others in between)

### Changing Actor Behaviour
You often need to provide different kinds of behaviour depending on the state of an actor, but this can involve using mutable variables which isn't great.

It's better to create a stateless actor which chooses to use one message handler or another.

`context.become(anotherHandler)` lets you replace the current message handler with a new message handler of type Receive. 

`context.become()` can take two parameters, the new message handler and a boolean for whether or not to discard the old message handler (defaults to true). If you pass `false`, the new message handler is added to a stack of previous message handlers.

`context.unbecome()` will pop the current message handler off the stack and go back to the previous one.

```Scala
// with mutable state variable - don't do this
class FussyKid extends Actor {  
  import FussyKid._  
  import Mom._  
  
  // internal state of the kid  
	var state = HAPPY  
	override def receive: Receive = {  
	  case Food(VEGETABLE) => state = SAD  
		case Food(CHOCOLATE) => state = HAPPY  
		case Ask(_) =>  
			if (state == HAPPY) sender() ! KidAccept  
			else sender() ! KidReject  
	}  
}

// stateless - do this
class StatelessFussyKid extends Actor {  
  import FussyKid._  
  import Mom._  
  
  override def receive: Receive = happyReceive  
  
	def happyReceive: Receive = {  
    case Food(VEGETABLE) => context.become(sadReceive, false) // change my receive handler to sadReceive  
		case Food(CHOCOLATE) => context.become(happyReceive, false)
		case Ask(_) => sender() ! KidAccept  
	}  
  
  def sadReceive: Receive = {  
    case Food(VEGETABLE) => context.become(sadReceive, false)  
    case Food(CHOCOLATE) => context.unbecome()  
    case Ask(_) => sender() ! KidReject  
	}  
}
```

`context.become()` and `context.unbecome()` let you change actor behaviour in response to messages.

Akka always uses the latest handler on top of the stack. If the stack is empty, it calls `receive` and use that message handler.