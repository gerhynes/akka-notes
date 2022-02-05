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