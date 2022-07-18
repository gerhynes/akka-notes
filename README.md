# Akka Notes

Sources:
- [Akka Essentials with Scala](https://www.udemy.com/course/akka-essentials/)
- [Akka HTTP with Scala](https://www.udemy.com/course/akka-http/)
- [Akka Persistence with Scala](https://www.udemy.com/course/akka-persistence/)

Akka is an open source toolkit and runtime that makes it easier to build concurrent, parallel and distributed applications on the JVM.

Akka provides:
-   Multi-threaded behavior without the use of low-level concurrency constructs like atomics or locks
-   Transparent remote communication between systems and their components
-   A clustered, high-availability architecture that elastically scales on demand

## Multithreading in Scala
On the JVM a new thread is created with a `Thread` constructor, which receives a `Runnable` object in which the `run` method does something.

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

The problem with threads is that they're unpredictable. If you run two threads it's completely unpredictable which order they'll run in. Also, different runs will produce different results.

The way you normally make threads safe is by adding `synchronized` blocks around the critical instructions. In a synchronized expression, two threads can't evaluate it at the same time.

You could also add `@volatile` to the private member, which would make reads atomic, preventing two threads from reading the value at the same time. This only works for primitive types such as `Int` however.

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
A `Future` represents a value which may or may not **currently** be available, but will be available at some point, or an exception if that value could not be made available.

For long running tasks, their value may or may not be currently available but will be available at some point in the future.

From a functional programming perspective, `Future` is a monadic construct, meaning it has functional primitives: `map`, `flatMap` and `filter`, as well as for comprehensions.

```Scala
// Global execution context
import scala.concurrent.ExecutionContext.Implicits.global

val future = Future {
	// long computation - on a different thread
	42
}

// callbacks
future.onComplete {
	case Success(42) => println("I found the meaning of life")
	case Failure(_) => println("Something happened with the meaning of life!")
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
Every Akka application starts with an `ActorSystem`.

An `ActorSystem` is a heavyweight data structure that controls a number of threads under the hood and allocates them to running actors.

You should have one `ActorSystem` per application unless you have a good reason to create more. The `ActorSystem`'s name must contain only alphanumeric characters and non-leading hyphens or underscores. Actors can be located by their actor system.

```Scala
val actorSystem = ActorSystem("firstActorSystem")

println(actorSystem.name) // firstActorSystem
```

- Actors are uniquely identified
- Messages are asynchronous
- Each actor has a unique way of processing the message
- Actors are (really) encapsulated

### Creating an Actor
You create an actor by creating a class that extends the `Actor` trait.

An actor needs a `receive` method. It takes no arguments and its return type is `PartialFunction[Any, Unit]` also aliased by the type `Receive`. This is a message handler object which is invoked when an actor processes a message.

You communicate with an actor by instantiating it (really an `ActorRef`) and then sending it a message.

You can't instantiate an actor by calling `new` but by invoking the `ActorSystem`. You pass in a `Props` object (typed with the actor type you want to instantiate) and the name for the actor.

The name restrictions for actors are the same as for the `ActorSystem`.

Instantiating an actor produces an `ActorRef`, the data structure that Akka exposes to you since you can't call the actual actor instance itself. You can only communicate with an actor via an `ActorRef`.

The method to invoke an `ActorRef` is `!`, also known as `tell` (Scala is very permissive about method naming).

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
You can instantiate an actor with constructor arguments using `Props` with an argument. Using `new` to instantiate an actor is legal once it's done inside `Props`' `apply()` method. `Props` is a data structure containing create and deploy information for the actor.

But the best practice is to declare a companion object and define a method that returns a `Props` object. You don't create actor instances yourself. Instead, the factory method creates `Props` with actor instances for you.

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

val alice = system.actorOf(Props[SimpleActor], "alice")
val bob = system.actorOf(Props[SimpleActor], "bob")
```

Messages can be almost any type but:
- they must be immutable
- they must be serializable

You need to enforce this principle since it currently can't be checked for at compile time.

In practice, you'll use case classes and case objects for almost all your message needs.

### Context
Actors have information about their context and about themselves.

Each actor has a member called `context`, a complex data structure with information about the environment the actor runs in.

`context.self` gives you access to the actors own `ActorRef` (the equivalent of `this` in the object-oriented world). 

You can use `self` to have an actor send a message to itself.

Actors can reply to messages using `context`. `context.sender()` returns an `ActorRef` which you can then use to send a message back.

For every actor, at any moment in time `context.sender()` contains the `ActorReference` of the last actor that sent a message to it. `context.sender()` can also be written as `sender()`.

Whenever an actor sends a message to another actor, they pass themselves as the sender.

The tell method receives the message and an implicit sender parameter, `Actor.noSender`, which is null. Since `self` is an implicit value, you usually omit it as a parameter.

```Scala
// under the hood
def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit

final val noSender: ActorRef = null

// inside SinmpleActor
case SayHiTo(ref) => ref ! "Hi!" // equivalent to (ref ! "Hi!")(self)
```

If there is no sender, the reply will go to a fake actor called  `dead letters` because `Actor.noSender` has a default value of null.

Actors can also forward messages to one another, keeping the orginal sender.

```Scala
case class WirelessPhoneMessage(content: String, ref: ActorRef)  
alice ! WirelessPhoneMessage("Hi", bob) // original sender is noSender.

// inside SimpleActor
case WirelessPhoneMessage(content, ref) => ref forward (content + "s") // keeps the original sender of the message
```

### Actor Recap
Every actor derives from the `Actor` trait, which has the abstract method `receive`.

`receive` returns a message handler object, which is retrieved by Akka when an actor receives a message.

This handler is invoked when the actor processes a message.

The `Receive` type is an alias of `PartialFunction[Any, Unit]`.

Actors need infrastructure in the form of an `ActorSystem`.

```Scala
val system = ActorSystem("AnActorSystem")
```

Creating an actor is done via the actor system, not via a new constructor. You need to call the `actorOf` factory method from the system, passing in a `Props` object ( a data structure with create/deploy information).

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

Actors are aware of the actor reference that last sent them a message using `sender()` and can use this to reply to messages: `sender() ! "well, hello there"`.

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

An actor has both a message handler and a message queue (mailbox). Whenever you send a message, it's enqueued in this mailbox.

An actor is a data structure, it's passive and needs a thread to run.

Akka spawns a small number of treads (100s) which can handle a large amount of actors (1000000s per GB heap).

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
You often need to provide different kinds of behaviour depending on the state of an actor, but this can involve using mutable variables, which isn't great.

It's better to create a stateless actor which chooses to use one message handler or another.

`context.become(anotherHandler)` lets you replace the current message handler with a new message handler of type `Receive`. 

`context.become()` can take two parameters, the new message handler and a boolean for whether or not to discard the old message handler (it defaults to `true`). If you pass `false`, the new message handler is added to a stack of previous message handlers.

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

Akka always uses the latest handler on top of the stack. If the stack is empty, it calls `receive` and uses that message handler.

If you want to rewrite a stateful actor to be a stateless actor, you need to rewrite its mutable state into the paramaters of the handlers you want to support.

```Scala
// a stateless counter
object Counter {  
	case object Increment  
	case object Decrement  
	case object Print  
}  
  
class Counter extends Actor {  
	import Counter._  
  
	override def receive: Receive = countReceive(0)  
	
	def countReceive(currentCount: Int): Receive = {  
	  case Increment =>  
			println(s"[countReceive($currentCount)] incrementing")  
      		context.become(countReceive(currentCount + 1))  
   	  case Decrement =>  
			println(s"[countReceive($currentCount)] decrementing")  
      		context.become(countReceive(currentCount - 1))  
      case Print => println(s"[countReceive($currentCount)] my current count is $currentCount")  
  }  
}  
  
import Counter._  
val counter = system.actorOf(Props[Counter], "myCounter")  
  
(1 to 5).foreach(_ => counter ! Increment)  
(1 to 3).foreach(_ => counter ! Decrement)  
counter ! Print
```

### Child Actors
Actors can create other actors using the actor's context: `context.actorOf(Props[Child], name)`.

This feature gives Akka the ability to create actor hierarchies where a parent actor supervises a child actor. 

A parent actor can create multiple children and each child actor can itself create child actors, creating a hierarchy tree.

```Scala
// create Parent and Child actors
object Parent {  
  case class CreateChild(name: String)  
  case class TellChild(message: String)  
}  

class Parent extends Actor {  
	import Parent._  
  
	override def receive: Receive = {  
		case CreateChild(name) =>  
			println(s"${self.path} creating child")  
	    	// create a new actor right HERE  
			val childRef = context.actorOf(Props[Child], name)  
	    	context.become(withChild(childRef))  
	}  
  
	def withChild(childRef: ActorRef): Receive = {  
    	case TellChild(message) => childRef forward message  
	}  
}  
  
class Child extends Actor {  
	override def receive: Receive = {  
    	case message => println(s"${self.path} I got: $message")  
  }  
}  
  
import Parent._  
  
val system = ActorSystem("ParentChildDemo")  
val parent = system.actorOf(Props[Parent], "parent")  
parent ! CreateChild("child")  
parent ! TellChild("hey Kid!")
```

Parent actors are supervised by guardian actors (top level actors).

Every Akka actor system has three guardian actors.
1. /system = system guardian
2. /user = user-level guardian
3. / = root-guardian

The root guardian manages both the system guardian and the user guardian.

#### Actor Selection
You can locate an actor in an actor hierarchy by creating an `ActorSelection`. You do this by passing the actor's path to `system.actorSelection()`. 

```Scala
val childSelection = system.actorSelection("/user/parent/child")
childSelection ! "I found you"
```

This `ActorSelection` is a wrapper over a potential `ActorRef` that you can use to send a messsage.

If the path is invalid, the `ActorSelection` will contain no actor and any message sent to it will be dropped (sent to dead letters).

### Dangers
Never pass mutable actor state, or the `this` reference, to child actors. This has the danger of breaking actor encapsulation because the child actor would suddenly have access to the internals of the parent actor.

If an actor's state is changed without the use of messages, it is extremely hard to debug. Calling a method directly on an actor also bypasses all the logic and security checks that the receive message handler would perform. 

When you use the `this` reference in a message, you're exposing yourself to method calls from other actors/threads, which exposes you to concurrency issues.

Never directly call methods on actors, only send messages.

This is called "closing over" mutable state or the `this` reference. Scala doesn't prevent this at compile time, so you need to make sure you maintain actor encapsulation.

```Scala
// do this
object CreditCard {
	case class AttachToAccount(bankAccountRef: ActorRef) // use ActorRef
	case object CheckStatus
}

// not this
object CreditCard {
	case class AttachToAccount(bankAccount: NaiveBankAccount) // don't use actor instance
	case object CheckStatus
}
```

### Actor Logging
It's useful to log information from actors while they're running. Distributed systems are hard to debug and when something crashes, logs are essential for figuring out what went wrong.

You can do this through explicit logging. 

`Logging` has an apply method which takes in an actor system and a logging source.

Logging is generally done on four levels:
1. debug - the most verbose
2. info - most benign messages
3. warn - e.g. messages being sent to `dead letters` and lost
4. error - an exception

```Scala
class SimpleActorWithExplicitLogger extends Actor {   
	val logger = Logging(context.system, this)  // this - this exact actor
  
	override def receive: Receive = {  
		case message => logger.info(message.toString)// log message 
	}  
}  
  
val system = ActorSystem("LoggingDemo")  
val actor = system.actorOf(Props[SimpleActorWithExplicitLogger])  
  
actor ! "Logging a simple message"
```

You can also extend an actor with the `ActorLogging` trait.

You can interpolate parameters into your log.

```Scala
class ActorWithLogging extends Actor with ActorLogging {  
	override def receive: Receive = {  
	  case (a, b) => log.info("Two things: {} and {}", a, b) // Two things: 2 and 3  
		case message => log.info(message.toString)  
  }  
}  
  
val simplerActor = system.actorOf(Props[ActorWithLogging])  
simplerActor ! "Logging a simple message by extending a trait"  
  
simplerActor ! (2, 3)
```

Logging is done asynchronously to minimize performance impact. Akka logging is done using actors itself. 

Logging doesn't depend on a particular logger implementation. The default logger just dumps things to standard output. You can insert another logger, such as SLF4J.

## Akka Configuration
Configuration in Akka is a series of name value pairs in a `.conf` file.

Configurations can be nested under namespaces.

```
myConfig = 42  
  
aBigConfiguration {  
  aParticularName = "akka2"  # aBigConfiguration.aParticularName  
  aNestedConfiguration {  
	  anotherNestedName = 56  
  }  
}
```

### Inline Configuration
You can pass a string to an actor system as a configuration using `ConfigFactory`.

All configurations in Akka will start with an Akka namespace.

```Scala
// inline configuration
val configString =  
	"""  
	| akka { 
	|   loglevel = "ERROR" 
	| } 
	""".stripMargin  
  
val config = ConfigFactory.parseString(configString)  
val system = ActorSystem("ConfigurationDemo", ConfigFactory.load(config))  
val actor = system.actorOf(Props[SimpleLoggingActor])  
  
actor ! "A message to remember"
```

### Using a Config File
When you create an actor system with no configuration, Akka automatically looks for a config file.

Config files are usualy stored in the `src/main/resources`.

```Scala
val defaultConfigFileSystem = ActorSystem("DefaultConfigFileDemo")  
val defaultConfigActor = defaultConfigFileSystem.actorOf(Props[SimpleLoggingActor])  
defaultConfigActor ! "Remember me"
```

### Separate Configs in the Same File
You can have multiple configurations in the same config file using seperate namespaces.

```Scala
// config file
mySpecialConfig {  
  akka {  
    loglevel = INFO  
  }  
}

// application
val specialConfig = ConfigFactory.load().getConfig("mySpecialConfig")  
val specialConfigSystem = ActorSystem("SpecialConfigDemo", specialConfig)  
val specialConfigActor = specialConfigSystem.actorOf(Props[SimpleLoggingActor])  
specialConfigActor ! "Remember me, I am special"
```

### Separate Configs in Another File
You can pass the relative path of another configuration file (relative to the resources directory) to load a separate config file.

```Scala
val separateConfig = ConfigFactory.load("secretFolder/secretConfiguration.conf")  
println(s"separate config log level: ${separateConfig.getString("akka.loglevel")}")
```

### Different Config File Formats
You can also use JSON or Properties files for Akka configurations.

```Scala
val jsonConfig = ConfigFactory.load("json/jsonConfig.json")  
println(s"json config: ${jsonConfig.getString("aJsonProperty")}")  
println(s"json config: ${jsonConfig.getString("akka.loglevel")}")  
  
val propsConfig = ConfigFactory.load("props/propsConfiguration.properties")  
println(s"properties config: ${propsConfig.getString("my.simpleProperty")}")  
println(s"properties config: ${propsConfig.getString("akka.loglevel")}")
```

## Testing
### Akka TestKit
Test classes usually are named after the class they test plus the `Spec` suffix. Test classes need to extend `TestKit`, which instantiates an actor system, as well as test-related interfaces: usually `ImplicitSender`, `WordSpecLike`, `BeforeAndAfterAll`.

The general structure of a test suite follows ScalaTest.

`afterAll` is used to tear down the test suite and actor system after all tests have run.

It's good practice to create a companion object for your tests. Here, you store the methods and values you're going to use in your tests.

You assert that an actor received a specific message using `expectMsg(message)`. 

If you expect no message to be returned, use `expectNoMessage(timeout)`. The timeout can be customized.

If you have stateful actors that you want to clear between tests, you create the actor inside a test suite and clear it at the start of each test.

`expectMsg()` tests basic equality. You can make more complex assertions about messages using `assert`. First, obtain the received message using `expectMsgType[TheMessageType]` and then use `assert` to assert something about the message.

`expectMsgAnyOf` lets you assert that the message received matches at least one of the provided messages.

`expectMsgAllOf` lets you assert that the messages received match all of the provided messages. 

`receiveN(numberOfMessages)` lets you store a number of received messages as a `Seq[Any]` for more complex assertions. 

`expectMsgPF()` lets you call a partial function on the message received, which gives you more power and granularity. As long as the partial function has a definition for the message that was received, the test will pass.

The `testActor` (a member of `TestKit`) is the sender/receiver of the messages used in tests. Because of the `ImplicitSender` trait, `testActor` is implicitly passed in as the sender of every message.

```Scala
import akka.actor.{Actor, ActorSystem, Props}  
import akka.testkit.{ImplicitSender, TestKit}  
import org.scalatest.{BeforeAndAfterAll, WordSpecLike}  
  
import scala.concurrent.duration._  
import scala.util.Random  
  
class BasicSpec extends TestKit(ActorSystem("BasicSpec"))  
  with ImplicitSender  
  with WordSpecLike  
  with BeforeAndAfterAll {  
  
  // setup  
  override def afterAll(): Unit = {  
    TestKit.shutdownActorSystem(system)  
  }  
  
  import BasicSpec._  
  
  "A simple actor" should {  
    "send back the same message" in {  
      val echoActor = system.actorOf(Props[SimpleActor])  
      val message = "hello, test"  
      echoActor ! message  
  
      expectMsg(message) // akka.test.single-expect-default  
    }  
  }  
  
  "A blackhole actor" should {  
    "send back no message" in {  
      val blackhole = system.actorOf(Props[Blackhole])  
      val message = "hello, test"  
      blackhole ! message  
  
      expectNoMessage(1 second)  
    }  
  }  
  
  // message assertions  
  "A lab test actor" should {  
    val labTestActor = system.actorOf(Props[LabTestActor])  
  
    "turn a string into uppercase" in {  
      labTestActor ! "I love Akka"  
      val reply = expectMsgType[String]  
  
      assert(reply == "I LOVE AKKA")  
    }  
  
    "reply to a greeting" in {  
      labTestActor ! "greeting"  
      expectMsgAnyOf("hi", "hello")  
    }  
  
    "reply with favorite tech" in {  
      labTestActor ! "favoriteTech"  
      expectMsgAllOf("Scala", "Akka")  
    }  
  
    "reply with cool tech in a different way" in {  
      labTestActor ! "favoriteTech"  
      val messages = receiveN(2) // Seq[Any]  
  
      // free to do more complicated assertions    
	}  
  
    "reply with cool tech in a fancy way" in {  
      labTestActor ! "favoriteTech"  
  
      expectMsgPF() {  
        case "Scala" => // only care that the PF is defined  
        case "Akka" =>  
      }  
    }  
  }  
}  
  
object BasicSpec {  
  
  class SimpleActor extends Actor {  
    override def receive: Receive = {  
      case message => sender() ! message  
    }  
  }  
  
  class Blackhole extends Actor {  
    override def receive: Receive = Actor.emptyBehavior  
  }  
  
  class LabTestActor extends Actor {  
    val random = new Random()  
  
    override def receive: Receive = {  
      case "greeting" =>  
        if (random.nextBoolean()) sender() ! "hi" else sender() ! "hello"  
      case "favoriteTech" =>  
        sender() ! "Scala"  
        sender() ! "Akka"  
      case message: String => sender() ! message.toUpperCase()  
    }  
  }  
}
```

### TestProbes
If you are testing an actor that interacts with other actors, you need entities that will hold the place for these other actors so that you can test the interaction. This is where `TestProbe`s come in.

A `TestProbe` is a special actor with some assertion capabilities. They can be instructed to send messages to a specific actor and reply with messages to the `TestProbe`'s last sender. They can also watch when other actors stop.

Here, if you register the follower actor with the leader actor, you can then test their interaction. 

```Scala
class TestProbeSpec extends TestKit(ActorSystem("TestProbeSpec"))  
  with ImplicitSender  
  with WordSpecLike  
  with BeforeAndAfterAll {  
  
  override def afterAll(): Unit = {  
    TestKit.shutdownActorSystem(system)  
  }  
  
  import TestProbeSpec._  
  
  "A leader actor" should {  
    "register a follower" in {  
      val leader = system.actorOf(Props[Leader])  
      val follower = TestProbe("follower")  
  
      leader ! Register(follower.ref)  
      expectMsg(RegistrationAck)  
    }  
  
    "send the work to the follower actor" in {  
      val leader = system.actorOf(Props[Leader])  
      val follower = TestProbe("follower")  
      leader ! Register(follower.ref)  
      expectMsg(RegistrationAck)  
  
      val workloadString = "I love Akka"  
      leader ! Work(workloadString)  
  
      // the interaction between the lead and the follow actor  
      follower.expectMsg(FollowerWork(workloadString, testActor))  
      follower.reply(WorkCompleted(3, testActor))  
  
      expectMsg(Report(3)) // testActor receives the Report(3)  
    }  
  
    "aggregate data correctly" in {  
      val leader = system.actorOf(Props[Lead])  
      val follower = TestProbe("follower")  
      leader ! Register(follower.ref)  
      expectMsg(RegistrationAck)  
  
      val workloadString = "I love Akka"  
      leader ! Work(workloadString)  
      leader ! Work(workloadString)  
  
      // in the meantime I don't have a follower actor  
      follower.receiveWhile() {  
		// `` indicate it must be this exact message
        case FollowerWork(`workloadString`, `testActor`) => 			
			follower.reply(WorkCompleted(3, testActor))  
      }  
  
      expectMsg(Report(3))  
      expectMsg(Report(6))  
    }  
  }  
}  
  
object TestProbeSpec {  
  // scenario  
  /*    
  word counting actor hierarchy leader-follower  
    send some work to the leader      
		- leader sends the follower the piece of work      
		- follower processes the work and replies to leader      
		- leader aggregates the result    
	leader sends the total count to the original requester   
  */  
  case class Work(text: String)  
  case class FollowerWork(text: String, originalRequester: ActorRef)  
  case class WorkCompleted(count: Int, originalRequester: ActorRef)  
  case class Register(followRef: ActorRef)  
  case object RegistrationAck  
  case class Report(totalCount: Int)  
  
  class Leader extends Actor {  
    override def receive: Receive = {  
      case Register(followerRef) =>  
        sender() ! RegistrationAck  
        context.become(online(followerRef, 0))  
      case _ => // ignore  
    }  
  
    def online(followerRef: ActorRef, totalWordCount: Int): Receive = {  
      case Work(text) => followerRef ! FollowerWork(text, sender())  
      case WorkCompleted(count, originalRequester) =>  
        val newTotalWordCount = totalWordCount + count  
        originalRequester ! Report(newTotalWordCount)  
        context.become(online(followerRef, newTotalWordCount))  
    }  
  }  
  
  // class Follower extends Actor ....  
}
```

### Timed Assertions
You can set timeboxes for tests, using `within(maxDuration)` or  `within(minDuration, maxDuration)` to assert that a message was received within a certain timeframe.

You can use `receiveWhile[Type](maxDuration, idle, numMessages) partialFunction` to capture a sequence of results. The partial function is run on each message received. You can then make assertions about this sequence of results.

`TestProbe`s don't listen to the configuration of the `within` block, they have their own configuration.

```Scala
class TimedAssertionsSpec extends TestKit(  
  ActorSystem("TimedAssertionsSpec", ConfigFactory.load().getConfig("specialTimedAssertionsConfig")))  
  with ImplicitSender  
  with WordSpecLike  
  with BeforeAndAfterAll {  
  
  override def afterAll(): Unit = {  
    TestKit.shutdownActorSystem(system)  
  }  
  
  import TimedAssertionsSpec._  
  
  "A worker actor" should {  
    val workerActor = system.actorOf(Props[WorkerActor])  
  
    "reply with the meaning of life in a timely manner" in {  
      within(500 millis, 1 second) {  
        workerActor ! "work"  
        expectMsg(WorkResult(42))  
      }  
    }  
  
    "reply with valid work at a reasonable cadence" in {  
      within(1 second) {  
        workerActor ! "workSequence"  
  
        val results: Seq[Int] = receiveWhile[Int](max=2 seconds, idle=500 millis, messages=4) {  
          case WorkResult(result) => result  
        }  
  
        assert(results.sum > 5)  
      }  
    }  
  
    "reply to a test probe in a timely manner" in {  
      within(1 second) {  
        val probe = TestProbe()  
        probe.send(workerActor, "work")  
        probe.expectMsg(WorkResult(42)) // timeout of 0.3 seconds based off application.conf  
      }  
    }  
  }  
}  
  
object TimedAssertionsSpec {  
  
  case class WorkResult(result: Int)  
  
  class WorkerActor extends Actor {  
    override def receive: Receive = {  
      case "work" =>  
        // simulate long computation  
        Thread.sleep(500)  
        sender() ! WorkResult(42)  
  
      case "workSequence" =>  
        val r = new Random()  
        for (_ <- 1 to 10) {  
          Thread.sleep(r.nextInt(50))  
          sender() ! WorkResult(1)  
        }  
    }  
  }  
}
```

### Intercepting Logs
Intercepting log messages is useful in integration tests, where it's hard to inject test probles into your actor architecture.

It would be difficult to do message-based testing in the below example.

`EventFilter` lets you check for log messages at specific levels: info, debug, warning or error. It also lets you check for expected exceptions: `EventFilter[RuntimeException](occurrences = 1)`.

You need to run your test code inside the `intercept` block. You also need to configure your logger so that `EventFilter` can intercept messages from it. `EventFilter` waits for a specifc amount of time, which you can also override in your configuration.

```Scala
// application.conf
interceptingLogMessages {
	akka {
		loggers = ["akka.testkit.TestEventListener"]
		test {  
  			filter-leeway = 5s  
		}
	}
}
```

```Scala  
class InterceptingLogsSpec extends TestKit(ActorSystem("InterceptingLogsSpec", ConfigFactory.load().getConfig("interceptingLogMessages")))  
  with ImplicitSender  
  with WordSpecLike  
  with BeforeAndAfterAll {  
  
  override def afterAll(): Unit = {  
    TestKit.shutdownActorSystem(system)  
  }  
  
  import InterceptingLogsSpec._  
  
  val item = "Akka course"  
  val creditCard = "1234-1234-1234-1234"  
  val invalidCreditCard = "0000-0000-0000-0000"  
  
  "A checkout flow" should {  
    "correctly log the dispatch of an order" in {  
      EventFilter.info(pattern = s"Order [0-9]+ for item $item has been dispatched.", occurrences = 1) intercept {  
        // our test code  
        val checkoutRef = system.actorOf(Props[CheckoutActor])  
        checkoutRef ! Checkout(item, creditCard)  
      }  
    }  
  
    "freak out if the payment is denied" in {  
      EventFilter[RuntimeException](occurrences = 1) intercept {  
        val checkoutRef = system.actorOf(Props[CheckoutActor])  
        checkoutRef ! Checkout(item, invalidCreditCard)  
      }  
    }  
  }  
}  
  
object InterceptingLogsSpec {  
  
  case class Checkout(item: String, creditCard: String)  
  case class AuthorizeCard(creditCard: String)  
  case object PaymentAccepted  
  case object PaymentDenied  
  case class DispatchOrder(item: String)  
  case object OrderConfirmed  
  
  class CheckoutActor extends Actor {  
    private val paymentManager = context.actorOf(Props[PaymentManager])  
    private val fulfillmentManager = context.actorOf(Props[FulfillmentManager])  
    override def receive: Receive = awaitingCheckout  
  
    def awaitingCheckout: Receive = {  
      case Checkout(item, card) =>  
        paymentManager ! AuthorizeCard(card)  
        context.become(pendingPayment(item))  
    }  
  
    def pendingPayment(item: String): Receive = {  
      case PaymentAccepted =>  
        fulfillmentManager ! DispatchOrder(item)  
        context.become(pendingFulfillment(item))  
      case PaymentDenied =>  
        throw new RuntimeException("I can't handle this anymore")  
    }  
  
    def pendingFulfillment(item: String): Receive = {  
      case OrderConfirmed => context.become(awaitingCheckout)  
    }  
  }  
  
  class PaymentManager extends Actor {  
    override def receive: Receive = {  
      case AuthorizeCard(card) =>  
        if (card.startsWith("0")) sender() ! PaymentDenied  
        else {  
          Thread.sleep(4000)  
          sender() ! PaymentAccepted  
        }  
    }  
  }  
  
  class FulfillmentManager extends Actor with ActorLogging {  
    var orderId = 43  
    override def receive: Receive = {  
      case DispatchOrder(item: String) =>  
        orderId += 1  
        log.info(s"Order $orderId for item $item has been dispatched.")  
        sender() ! OrderConfirmed  
    }  
  }  
}
```

### Synchronous Testing
Synchronous testing handles all messages in the calling thread. This doesn't need the `TestKit` infrastructure but you still need an implicit `system` value.

Sending a message to a `TestActorRef` happens in the calling thread. When you send a message to it, the thread will only return after the message has been processed. You can then do assertions knowing that the message has been received.

`TestActorRef` lets you access the underlying actor and make assertions about its state. It also lets you invoke the `receive` handler on the underlying actor.

You can also mimic this single thread behaviour using the calling thread dispatcher. This means that the communication with the actor happens on the calling thread. When you call `expectMsg`, the `TestProbe` will already have received the message.

This testing style doesn't work well with actors that have to be asynchronous to work properly, such as persistent actors.

```Scala
class SynchronousTestingSpec extends WordSpecLike with BeforeAndAfterAll {  
  
  implicit val system = ActorSystem("SynchronousTestingSpec")  
  
  override def afterAll(): Unit = {  
    system.terminate()  
  }  
  
  import SynchronousTestingSpec._  
  
  "A counter" should {  
    "synchronously increase its counter" in {  
      val counter = TestActorRef[Counter](Props[Counter])  
      counter ! Inc // counter has ALREADY received the message  
  
      assert(counter.underlyingActor.count == 1)  
    }  
  
    "synchronously increase its counter at the call of the receive function" in {  
      val counter = TestActorRef[Counter](Props[Counter])  
      counter.receive(Inc)  
      assert(counter.underlyingActor.count == 1)  
    }  
  
    "work on the calling thread dispatcher" in {  
      val counter = system.actorOf(Props[Counter])  
      val probe = TestProbe()  
  
      probe.send(counter, Read)  
      probe.expectMsg(Duration.Zero, 0) // probe has ALREADY received the message 0  
    }  
  }  
}  
  
object SynchronousTestingSpec {  
  case object Inc  
  case object Read  
  
  class Counter extends Actor {  
    var count = 0  
  
    override def receive: Receive = {  
      case Inc => count += 1  
      case Read => sender() ! count  
    }  
  }  
}
```

## Fault Tolerance
### Starting, Stopping and Watching Actors
Starting actors is relatively simple. Use `system.actorOf` or `context.actorOf`.

Stopping actors is more complicated.

A parent actor can use `context.stop(childRef)`. This is asynchronous and non-blocking, an actor may continue to receive messages until actually stopped. If you call `context.stop(self)` on a parent actor, it will also stop its children, waiting for the children to stop before stopping the parent actor.

You can also use special messages. If you send `PoisonPill` to an actor, it will cause it to stop. `Kill` will stop an actor and make it throw an `ActorKilledException`. These are handled by a separate receive handler behind the scenes, so you can't catch a `PoisonPill` and ignore it, for example.

Death watch is a mechanism for being notified when an actor dies. `context.watch(actorRef)` registers an actor to watch for the death of another actor, such as a child. The actor will receive a `Terminated` message with a reference to the actor that has just been stopped. You will still received the `Terminated` message if the actor you're watching is already dead when you register for its death watch. `context.watch()` can be invoked more than once on a number of actors. `context.unwatch()` lets you unsubscribe.

### Actor Lifecycle
Some important distinctions.

An **actor instance** 
- has methods
- may have internal state

An **actor reference** (incarnation)
- created with `actorOf`
- has mailbox and can receive messages
- contains one actor instance
- contains a UUID

An **actor path**
- may or may not have an `ActorRef` inside

Actors can be subject to a number of actions. They can be
- started
- suspended
- resumed
- restarted
- stopped

Starting creates a new `ActorRef` with a UUID at a given path

Suspending means the actor reference may still enqueue but will not process more messages

Resuming means the actor reference will continue processing more messages

Restarting proceeds in the following steps:
- the actor reference is suspended
- the actor instance is swapped
	- the old instance calls `preRestart()`
	- the actor instance is replaced
	- the new instance calls `postRestart()`
- the actor reference is resumed

Any internal state is destroyed on restart.

Stopping frees the actor reference within a path.
- the actor instance calls `postStop()`
- all watching actors receive `Terminated(ref)`

After stopping, another actor may be created at the same path. This has a different UUID, and so a different `ActorRef`. As a result of stopping, all the messages currently enqueued on the actor reference will be lost.

### Supervision
It's fine if actors fail. Parent actors must decide how to handle their child actors' failure.

When an actor fails:
- it suspends its children
- it sends a special message to its parent via a dedicated message queue

The parent can decide to:
-  resume the actor
-  restart the actor (default) - clears internal state and stops children
-  stop the actor
-  escalate and fail itself

Parent actors decide on their children's failure with a supervision strategy. The `OneForOneStrategy` is applied just to the child that fails, the `AllForOneStrategy` is applied to all the children, not just the one that failed. 

The supervisor strategy takes parameters for how many times and the time window during which the strategy is being retried, after which the actor is simply stopped. The other argument is a decider, a `PartialFuntion[Throwable, Directive]`.

```Scala
override val supervisorStrategy = OneForOneStrategy(maxNumOfRetries = 10, withinTimeRange = 1 minute) {
	case e: IllegalArgumentException => Restart
	// other cases
}
```

The sealed trait `Directive` has case objects which are the options an actor can take if it receives the signal for failure from one of its children.

```Scala
class Supervisor extends Actor {
	override val supervisorStrategy: SupervisorStrategy = OneForOneStrategy() { 
	  case _: NullPointerException => Restart  
	  case _: IllegalArgumentException => Stop  
	  case _: RuntimeException => Resume  
	  case _: Exception => Escalate  
	}
}	
```

This results in actors being fault-tolerant and self-healing.

### The Backoff Supervisor Pattern
The backoff supervisor pattern aims to solve the problem of repeated restarts of actors, especially in the context of the actors interacting with an external resource such as a database.

Restarting actors immediately might do more harm than good, for example if a database goes down and comes back up, the build up of concurrent requests could cause it to go down again. 

The backoff supervisor pattern introduces exponential delays and randomness between attempts to rerun a supervision strategy. You can decide to trigger the backoff on failure or on stop and configure minimum and maximum delays between retries as well as a randomness factor. 

```Scala
val simpleSupervisorProps = BackoffSupervisor.props(  
  Backoff.onFailure(  
    Props[FileBasedPersistentActor],  
    "simpleBackoffActor",  
    3 seconds, // then 6s, 12s, 24s  
    30 seconds, // time range 
    0.2 // randomness factor 
  )  
)

val stopSupervisorProps = BackoffSupervisor.props(  
  Backoff.onStop(  
    Props[FileBasedPersistentActor],  
    "stopBackoffActor",  
    3 seconds,  
    30 seconds,  
    0.2  
  ).withSupervisorStrategy(  
    OneForOneStrategy() {  
      case _ => Stop  
    }  
  )  
)

val simpleBackoffSupervisor = system.actorOf(simpleSupervisorProps, "simpleSupervisor")  

val stopSupervisor = system.actorOf(stopSupervisorProps, "stopSupervisor")
```

### Schedulers and Timers
Schedulers and timers aim to solve the problem of runnng some code at a defined point in the future, maybe repeatedly.

Like `Futures`, scheduling has to happen on a thread and so you need an `ExecutionContext`. Import `system.dispatcher`.

`system.scheduler.scheduleOnce(delay)` lets you set a delay on the execution of the scheduled action.

`system.scheduler.schedule(delay, interval)` lets you specify a delay and interval for repeating actions. 

Schedules like this are of type `Cancellable` and can be cancelled by calling the `cancel` method on the schedule.

- Don't use unstable references inside scheduled actions.
- All scheduled tasks execute when the system is terminated.
- Schedulers are not the best at precise and long-term planning.

Actors can schedule messages to themselves using the `Timers` trait. This gives the actor access to the `startSingleTimer` and `startPeriodicTimer` methods. 

These methods take as parameters: an identifying `TimerKey` (a case object), the message the actor will send to itself, and the initial delay/interval.

If you use the same `TimerKey` as was associated wth another timer, the previous timer is cancelled. `timers.cancel(TimerKey)` will free up the timer associated wiith that `TimerKey`.

```Scala
case object TimerKey  
case object Start  
case object Reminder  
case object Stop  

class TimerBasedHeartbeatActor extends Actor with ActorLogging with Timers { 
  timers.startSingleTimer(TimerKey, Start, 500 millis)  
  
  override def receive: Receive = {  
    case Start =>  
      log.info("Bootstrapping")  
      timers.startPeriodicTimer(TimerKey, Reminder, 1 second)  
    case Reminder =>  
      log.info("I am alive")  
    case Stop =>  
      log.warning("Stopping!")  
      timers.cancel(TimerKey)  
      context.stop(self)  
  }  
}  
  
val timerHeartbeatActor = system.actorOf(Props[TimerBasedHeartbeatActor], "timerActor")  
system.scheduler.scheduleOnce(5 seconds) {  
  timerHeartbeatActor ! Stop  
}
```

### Routers
Routers are useful when you want to delegate or spread work between multiple actors of the same kind. Routers are usually middle-level actors that foward messages to other actors.

Less commonly, you can manually create a router with `Router(RouterLogic, routees)`.

The supported options for routing logic are:
- round robin
- random
- smallest mailbox
- broadcast
- scatter-gather-first
- tail-chopping
- consistent-hashing

The more common method for creating a router is to create a Pool Router, a router actor wth its own children.

```Scala
class Follower extends Actor with ActorLogging {  
  override def receive: Receive = {  
    case message => log.info(message.toString)  
  }  
}

// Creates a leader actor and 5 followers under itself
val poolLeader = system.actorOf(RoundRobinPool(5).props(Props[Follower]), "simplePoolLeader")
```

Routing can be configured in `application.conf`.

```
// application.conf
routersDemo {
	akka {  
	  actor.deployment {  
		/poolLeader2 {  
		  router = round-robin-pool  
		  nr-of-instances = 5  
		}   
	}
}
```

```Scala
val system = ActorSystem("RoutersDemo", ConfigFactory.load().getConfig("routersDemo"))

// Needs to have same name as path used in config
val poolLeader2 = system.actorOf(FromConfig.props(Props[Follower]), "poolLeader2")
```

The router can access actors that were created elsewhere in your application. Their paths can be defined in `application.conf`.

```
// application.conf
routersDemo {
	akka {  
	  actor.deployment {  
		/groupLeader2 {  
		  router = round-robin-group  
		  routees.paths = ["/user/follower_1","/user/follower_2","/user/follower_3","/user/follower_4","/user/follower_5"]  
		}  
	  }  
	}
}
```

```Scala
// In another part of the application
val followerList = (1 to 5).map(i => system.actorOf(Props[Follower], s"follower_$i")).toList  
  

// Option1 - need to get their paths  
val followerPaths = followerList.map(followerRef => followerRef.path.toString)  
val groupLeader = system.actorOf(RoundRobinGroup(followerPaths).props())

// Option2 - paths defined in config
val groupLeader2 = system.actorOf(FromConfig.props(), "groupLeader2")
```

The `Broadcast(message)` message will be sent to every routed actor regardless of the routing strategy.

`PoisonPill` and `Kill` are not routed. They are handled by the routing actor.

`AddRoutee`, `RemoveRoutee`, `GetRoutee` are also handled only by the routing actor.

### Dispatchers
Dispatchers are in charge of how messages are delivered and handled within an actor system.

Every dispatcher has an `executor` which handles which messages are handled on which thread. You can configure how many threads are allocated to an `executor`, as well as throughput, which is the number of messages a dispatcher can handle for one actor before that thread moves to another actor.

```
// application.conf
my-dispatcher {  
  type = Dispatcher # PinnedDispatcher, CallingThreadDispatcher  
  executor = "thread-pool-executor"  
  thread-pool-executor {  
    fixed-pool-size = 3  
  }  
  throughput = 30  
}
```

```Scala
val system = ActorSystem("DispatchersDemo") // , ConfigFactory.load().getConfig("dispatchersDemo")

val actors = for (i <- 1 to 10) yield system.actorOf(Props[Counter].withDispatcher("my-dispatcher"), s"counter_$i")
```

Dispatchers implement the `ExecutionContext` trait, which lets them schedule actions and run `Future`s on threads.

```Scala
class DBActor extends Actor with ActorLogging {   
  implicit val executionContext: ExecutionContext = context.system.dispatchers.lookup("my-dispatcher")  
  
  override def receive: Receive = {  
    case message => Future {  
      // wait on a resource  
      Thread.sleep(5000)  
      log.info(s"Success: $message")  
    }  
  }  
}
```

There are multiple types of dispatchers, with `Dispatcher` being the most often used. It is based on an executor service, which binds an actor to a thread pool. 

There are also `PinnedDispatcher`s, which bind each actor to a thread pool of exactly one thread, and `CallingThreadDispatcher`s, which ensure that all invocations and all communications wth an actor happen on the calling thread, whatever that thread is.

### Mailboxes
Mailboxes are the data structures in the `ActorRef` that control how messages are stored.

Normally mailboxes enqueue messages sent to actors in a regular queue. You can however create a custom mailbox to enforce priority. 

Custom mailboxes need a settings parameter of type `ActorSystem.Settings` and config of type `Config`. If you extend the trait `UnboundedPriorityMailbox`, it takes a `PriorityGenerator` parameter, which takes a partial function from a message to a priority. 

```Scala
class SupportTicketPriorityMailbox(settings: ActorSystem.Settings, config: Config)  
  extends UnboundedPriorityMailbox(  
    PriorityGenerator {  
      case message: String if message.startsWith("[P0]") => 0  
      case message: String if message.startsWith("[P1]") => 1  
      case message: String if message.startsWith("[P2]") => 2  
      case message: String if message.startsWith("[P3]") => 3  
      case _ => 4  
    })

val supportTicketLogger = system.actorOf(Props[SimpleActor].withDispatcher("support-ticket-dispatcher"))
```

## Akka Patterns
### Stashing Messages
Stashing allows actors to put aside messages for later because they can't/shouldn't process them right now. 

When the time is right (usually when the actor changes behavour with `context.become`/`context.unbecome`), the stashed messages are prepended to the mailbox and processed.

You stash a message by having the actor extend the `Stash` trait and then use the `stash()` method to stash messages and `unstashAll()` to unstash them when you switch message handler.

```Scala
class ResourceActor extends Actor with ActorLogging with Stash {  
  private var innerData: String = ""  
  
  override def receive: Receive = closed  
  
  def closed: Receive = {  
    case Open =>  
      log.info("Opening resource")  
      // unstashAll when you switch the message handler  
      unstashAll()  
      context.become(open)  
    case message =>  
      log.info(s"Stashing $message because I can't handle it in the closed state")  
      // stash away what you can't handle  
      stash()  
  }  
  
  def open: Receive = {  
    case Read =>  
      // do some actual computation  
      log.info(s"I have read $innerData")  
    case Write(data) =>  
      log.info(s"I am writing $data")  
      innerData = data  
    case Close =>  
      log.info("Closing resource")  
      unstashAll()  
      context.become(closed)  
    case message =>  
      log.info(s"Stashing $message because I can't handle it in the open state") 
      stash()  
  }  
}
```

- There are potential memory bounds on stash.
- There are potential mailbox bounds when unstashing.
- You can't stash the same message twice. It will throw an exception.
- The `Stash` trait overrides `preRestart` and so must be mixed-in last.

### The Ask Pattern
The ask pattern lets you communicate with actors when you expect a single response.

```Scala
val askFuture = actor ? Read("Some message")
```

Calling ask (`?`) on an actor will return a `Future` with the potential response that you might get from the actor. As such, you need both an implicit `timeout` and an `ExecutionContext`.

You can process the `Future` using `onComplete` or you can pipe its contents to another `actorRef`.

```Scala
askFuture.onComplete {
	case ...
}

askFuture.mapto[String].pipeTo(otherActor)
```

Never call methods on the actor instance or access mutable state in callbacks or  `onComplete`. You can break the actor encapsulation. Save the original sender before handling the completed future with `onComplete` to avoid closing over the actor instance or mutable state. 

```Scala
case class RegisterUser(username: String, password: String)  
case class Authenticate(username: String, password: String)  
case class AuthFailure(message: String)  
case object AuthSuccess  

object AuthManager {  
  val AUTH_FAILURE_NOT_FOUND = "username not found"  
  val AUTH_FAILURE_PASSWORD_INCORRECT = "password incorrect"  
  val AUTH_FAILURE_SYSTEM = "system error"  
}  

class AuthManager extends Actor with ActorLogging {  
  import AuthManager._  
  
  implicit val timeout: Timeout = Timeout(1 second)  
  implicit val executionContext: ExecutionContext = context.dispatcher  
  
  protected val authDb = context.actorOf(Props[KVActor])  
  
  override def receive: Receive = {  
    case RegisterUser(username, password) => authDb ! Write(username, password) 
    case Authenticate(username, password) => handleAuthentication(username, password)  
  }  
  
  def handleAuthentication(username: String, password: String) = {  
    val originalSender = sender()  
    // ask the actor  
    val future = authDb ? Read(username)  
    // handle the future  
    future.onComplete {  
      // NEVER CALL METHODS ON THE ACTOR INSTANCE OR ACCESS MUTABLE STATE IN ONCOMPLETE.      
	 // avoid closing over the actor instance or mutable state      
	  case Success(None) => originalSender ! AuthFailure(AUTH_FAILURE_NOT_FOUND) 
      case Success(Some(dbPassword)) =>  
        if (dbPassword == password) originalSender ! AuthSuccess  
        else originalSender ! AuthFailure(AUTH_FAILURE_PASSWORD_INCORRECT)  
      case Failure(_) => originalSender ! AuthFailure(AUTH_FAILURE_SYSTEM)  
    }  
  }  
}

class PipedAuthManager extends AuthManager {  
  import AuthManager._  
  
  override def handleAuthentication(username: String, password: String): Unit = {  
    // ask the actor  
    val future = authDb ? Read(username) // Future[Any]  
    // process the future until you get the responses you will send back    
	val passwordFuture = future.mapTo[Option[String]] // Future[Option[String]] 
    val responseFuture = passwordFuture.map {  
      case None => AuthFailure(AUTH_FAILURE_NOT_FOUND)  
      case Some(dbPassword) =>  
        if (dbPassword == password) AuthSuccess  
        else AuthFailure(AUTH_FAILURE_PASSWORD_INCORRECT)  
    } // Future[Any] - will be completed with the response I will send back  
  
    // pipe the resulting future to the actor you want to send the result to    
	  /*      
	  	When the future completes, send the response to the actor ref in the arg list.     
	  */    
	  responseFuture.pipeTo(sender())  
  }  
}
```

### Finite State Machines
Finite state machines are an alternative when `context.become` gets too complicated.

An finite state machine is an actor that at any point has some state and data. A state is an instance of a class (an object), as is the data. As the name implies, a finite state machine has a finite number of states that it can exist in.

In rection to an event, both the state and the data might be replaced with different values.

To create an FSM actor, first define the states and the data of the actor using case objects.

Have the actor extend the `FSM` trait, typed with the states and data.

In an FSM actor, you don't handle messages directly with a `receive` handler. When an FSM receives a message, it triggers an event. The event contains the message and the data that currently exists in the actor.

Unlike regular actors, the job of the FSM actor is to handle states and events, not messages.

Use `startWith(state, data)` to set the initial state and data.

You handle state changes by calling the `when(state)` method. This takes as a second parameter list a partial function from an event to another state. 

When the actor receives a message, the appropriate `when` clause is triggered, based off its current state. Inside `when`'s partial function, one of these event cases is triggered. This handles the message and the data you currently have, and may update the state and data.

Use `goto(state) using data` to change to a different state while keeping the current data. 

```Scala
when(state) {
	case Event(message, currentData) =>
		// handle the message
		goto(newState) with updatedData
}
```

You can also use `stay()` to stay in the current state. You can combine `stay()` with `using` to stay in the current state and hold onto the current data.

Use `whenUnhandled { case ... }`, which takes a partial function between events and states, to handle events that aren't being handled elsewhere.

You can use `onTransition`, which takes a partial function between a tuple of states and `Unit`, to perform a side effect when switching from one state to another.

Finally, calling the `initialize()` method starts the whole chain of message handlers.

Akka FSM makes is easy to set schedules for your states. You add a `timeout` parameter in the `when` method: `when(state, stateTimeout = 1 second)`. If the timeout elapses without an event being triggered, Akka will trigger an event automatically with a `StateTimeout`. You can handle this in the partial function with `case Event(StateTimeout, data) =>  // do something`.

```Scala
trait VendingState  
case object Idle extends VendingState  
case object Operational extends VendingState  
case object WaitForMoney extends VendingState  
  
trait VendingData  
case object Uninitialized extends VendingData  
case class Initialized(inventory: Map[String, Int], prices: Map[String, Int]) extends VendingData  
case class WaitForMoneyData(inventory: Map[String, Int], prices: Map[String, Int], product: String, money: Int, requester: ActorRef) extends VendingData  
  
class VendingMachineFSM extends FSM[VendingState, VendingData] {  
  // we don't have a receive handler  
  
  // an EVENT(message, data)  
  /*    
  	state, data    
	event => change state and data  
    
	initially
	state = Idle    
	data = Uninitialized  
	
    event(Initialize(Map(coke -> 10), Map(coke -> 1)))      
		=>      
		state = Operational      
		data = Initialized(Map(coke -> 10), Map(coke -> 1))  
  
    event(RequestProduct(coke))      
		=>      
		state = WaitForMoney      
		data = WaitForMoneyData(Map(coke -> 10), Map(coke -> 1), coke, 0, R)  
    
	event(ReceiveMoney(2))      
		=>      
		state = Operational      
		data = Initialized(Map(coke -> 9), Map(coke -> 1))  
   */  
	
  // initialize state and data
  startWith(Idle, Uninitialized)  
  
  // handle Idle state
  when(Idle) {  
    case Event(Initialize(inventory, prices), Uninitialized) =>  
      goto(Operational) using Initialized(inventory, prices)  
    // equivalent to context.become(operational(inventory, prices))  
    case _ =>  
      sender() ! VendingError("MachineNotInitialized")  
      stay() // stay in current state
  }  
  
  when(Operational) {  
    case Event(RequestProduct(product), Initialized(inventory, prices)) =>  
      inventory.get(product) match {  
        case None | Some(0) =>  
          sender() ! VendingError("ProductNotAvailable")  
          stay()  
        case Some(_) =>  
          val price = prices(product)  
          sender() ! Instruction(s"Please insert $price dollars")  
		  // change to WaitForMoney state, keep data
          goto(WaitForMoney) using WaitForMoneyData(inventory, prices, product, 0, sender())  
      }  
  }  
  
  when(WaitForMoney, stateTimeout = 1 second) {  
    case Event(StateTimeout, WaitForMoneyData(inventory, prices, product, money, requester)) =>  
      requester ! VendingError("RequestTimedOut")  
      if (money > 0) requester ! GiveBackChange(money)  
      goto(Operational) using Initialized(inventory, prices)  
  
    case Event(ReceiveMoney(amount), WaitForMoneyData(inventory, prices, product, money, requester)) =>  
      val price = prices(product)  
      if (money + amount >= price) {  
        // user buys product  
        requester ! Deliver(product)  
        // deliver the change  
        if (money + amount - price > 0) requester ! GiveBackChange(money + amount - price)  
        // updating inventory  
        val newStock = inventory(product) - 1  
        val newInventory = inventory + (product -> newStock)  
  
        goto(Operational) using Initialized(newInventory, prices)  
      } else {  
        val remainingMoney = price - money - amount  
        requester ! Instruction(s"Please insert $remainingMoney dollars")  
  
        stay() using WaitForMoneyData(inventory, prices, product, money + amount, requester)  
      }  
  }  
  
  whenUnhandled {  
    case Event(_, _) =>  
      sender() ! VendingError("CommandNotFound")  
      stay()  
  }  
  
  onTransition {  
    case stateA -> stateB => log.info(s"Transitioning from $stateA to $stateB") 
  }  
  
  initialize()  
}
```

## Akka Streams
Treating data as a stream of elements instead of in its entirety is useful because it matches the way computers send and receive data (for example via TCP), but it is often also a necessity because data sets frequently become too large to be handled as a whole.

Actors can be seen as dealing with streams as well: they send and receive series of messages in order to transfer data from one place to another.

The Akka Streams API exists to safely formulate stream processing setups with actors.

Akka Streams require an `ActorSystem` and an `ActorMaterializer`.

An `ActorMaterializer` is a special kind of object that allocates the right resources for constructing Akka Sreams components.

A `Source` publishes elements, a `Sink` consumes elements, and between them you can put a `Flow`.

You construct an Akka Stream by connecting these components together. The connected components don't do anything until you connect them in a `runnableGraph` and call its `run` method. 

The `run` method takes `ActorMaterializer` as an implicit value, and the `ActorMaterializer` handles the stream's resources.

```Scala
implicit val system = ActorSystem("AkkaStreamsRecap")  
implicit val materializer = ActorMaterializer() 
  
val source = Source(1 to 100)  
val sink = Sink.foreach[Int](println)  
val flow = Flow[Int].map(x => x + 1)  
  
val runnableGraph = source.via(flow).to(sink)  
runnableGraph.run()
```

### Materialized Values
You can extract meaningful values from the running of a graph. This is called materialization.

This example will produce a `Future` containing the sum of all the elements that flow through this graph. The `Future` will be completed when the stream terminates.

When you materialize a `runnableGraph`, you materialize every component that composes that graph. You can choose which materialized value you care about.

```Scala
val simpleMaterializedValue = runnableGraph.run() // materialization  
  
// MATERIALIZED VALUE  
val sumSink = Sink.fold[Int, Int](0)((currentSum, element) => currentSum + element)  
val sumFuture = source.runWith(sumSink)

sumFuture.onComplete {  
	case Success(sum) => println(s"The sum of all the numbers from the simple source is: $sum")  
	case Failure(ex) => println(s"Summing all the numbers from the simple source FAILED: $ex")  
}
```

Materializing a graph means materializing **all** the components  

A materialized value can be **anything at all**, such as a `Future` or a HTTP connection. It may or may not have anything to do with the actual elements that flow through the graph.

### Backpressure
Akka streams operate as a response to demand from downstream consumers. 

Backpressure lets you slow down a producer if the consumers are slow and don't issue any demand. Backpressure happens transparently at runtime and components can choose how to respond in between a fast upstream and a slow downstream.

Backpressure actions include:
- buffer elements
- apply a strategy in case the buffer overflows
- drop elements
- drop the buffer
- fail the entire stream

```Scala
val bufferedFlow = Flow[Int].buffer(10, OverflowStrategy.dropHead)  
  
source.async  
 .via(bufferedFlow).async  
 .runForeach { e =>  
		// a slow consumer  
		Thread.sleep(100)  
		println(e)  
  }
```

## Akka Persistence
### Event Sourcing
Traditionally, databases reflect the state of a class. This has certain problems.

- How do you query a previous state?
- How did you arrive at this state?

Applications that need to address this include: 
- tracing orders in an online store
- transaction history in a bank
- chat messages
- document versioning (like Dropbox)

For example, if you track all the events around an order, you get a richer description.
- CreateOrder(details)
- AllocateInventory(details)
- DispatchOrder
- DeliverOrder(success)
- ReturnOrder(reasons)
- CancelOrder(reasons)
- RefundOrder(amount)

Instead of storing current state, you store events. You can always recreate the current state by replaying these events.

Pros
- high performance: events are append only
- avoids relational stores and Object-relational Mapping (ORM) entirely
- full trace of every state
- fits the Akka actor model (every event is a message)

Cons
- querying a state is potentially expensive
- potential performance issues with long-lived entities
- data model is subject to change
- new mental model

### Persistent Actors
Persistent actiors can do everything normal actors can
- send and receive messages
- hold internal state
- run in parallel with many other actors

They also have extra capabilities
- have a unique persistence ID
- can persist events to a long-term store (or "journal")
- can recover state by replaying events from the store

When an actor receives a message (command)
- it can asynchronously persist an event to the store
- after the event is persisted, it changes its internal state and/or sends messages to other actors

When a persistent actor starts/restarts
- it replays all the events associated with its persistence ID
- commands received during the recovery stage are stashed until the recovery has completed.

#### A Persistent Actor
To create a persistent actor, create an actor that extends the `PersistentActor`trait.

Implement three methods: `persistenceId`, `receiveCommand` and `receiveRecover`.

`persistenceId` is how the events persisted by this actor will be identified in the persistence store. It should be unique per actor.

`receiveCommand` is the "normal" `receive` method. It's the hanlder that will be called when you send a normal message to this actor.

`receiveRecover` is the handler that's called on recovery. The actor will query the persistence store for all the events associated with this actor's `persistenceId` and replay them in the order in which they were written. They'll be sent to the actor as simple messages but will be handled by `receiveRecover`.

As with any normal actor, you need a system and need to initialize the actor with `system.actorOf`.

When a persistent actor receives a command:
1. you create an event to persist to the store
2. you persist the event, passing in a callback that will get triggered once the event is written
3. update the actor's state when the event has persisted

`persist` takes as parameters an event and the handler that will be called when the event has persisted.

`receiveRecover` follows the logic of the steps in `receiveCommand`. When the actor next starts, it will receive all the messages it previously received in the same order in which they were originally persisted.

Persisting is asynchronous and non-blocking. It will be executed at some point in the future. The handler will also be executed at some point in the future after the event has been successfully persisted.

It is safe to access mutable state in the handler since Akka Persistence guarantees that no other threads are accessing the actor during a callback. During the callback, you can also access the original sender of the command.

There is a time gap between the original call to `persist` and the callback. Messages sent to this actor during this time gap are stashed.

A persistent actor is not obliged to persist an event from every message. It can handle messages just like a normal actor.

#### Persistence Failures
There are two main types of persistence error: 
- the call to `persist` throws an error
- the journal implementation fails to persist a message

Persistent actors have callbacks in case persisting fails.

`onPersistFailure` is called if the `persist` call fails. It has access to the cause (a `Throwable`), the event, and the sequence number of the event from the journal's point of view. In this case, the actor will be unconditionally **stopped** (ragardless of the supervision strategy), because you don't know if the event was persisted or not. It's a good practice to start the actor again after a while using the Backoff supervisor pattern.

```Scala
override def onPersistFailure(cause: Throwable, event: Any, seqNr: Long): Unit = {  
	log.error(s"Fail to persist $event because of $cause")  
    super.onPersistFailure(cause, event, seqNr)  
}  
```

`onPersistRejected` is called if the journal throws an exception while persisting the event. In this case the actor is **resumed**, because you know for sure that the event wasn't persisted and the actor's state wasn't corrupted.

```Scala
override def onPersistRejected(cause: Throwable, event: Any, seqNr: Long): Unit = {  
	log.error(s"Persist rejected for $event because of $cause")  
    super.onPersistRejected(cause, event, seqNr)  
}  
```

You can persist multiple events at the same time atomically using `persistAll`. It takes a sequence of events and a handler to be called when each event has persisted.

When the actor receives a command to persist multiple events:
1. create the events
2. persist all the events
3. update the actor state after each event is persisted

Never call `persist` or `persistAll` from `Future`s. You would risk breaking actor encapsulation because the actor thread is free to process messages while you are persisting. If the normal actor thread also called `persist`, you'd have two threads persisting events simultaneously, which risks corrupting the actor state.

#### Shutting Down Persistent Actors 
Normal actors are relatively straightforward to shut down wth `Poison Pill` or `Kill`. If you do this to a persistent actor, you risk killing it before it is done persisting (because messages are stashed during persisting and `Poison Pill` is handled in a separate mailbox).

You should define a custom shutdown message. This message will be put into the normal mailbox, which guarantees it is handled after all the other messages that have been received.

```Scala
case Shutdown =>
	context.stop(self)
```

```Scala
object PersistentActors extends App {   
  // Commands  
  case class Invoice(recipient: String, date: Date, amount: Int)  
  case class InvoiceBulk(invoices: List[Invoice])  
  
  // Special messsages  
  case object Shutdown  
  
  // Events  
  case class InvoiceRecorded(id: Int, recipient: String, date: Date, amount: Int)  
  
  class Accountant extends PersistentActor with ActorLogging { 
    var latestInvoiceId = 0  
    var totalAmount = 0  
  
    override def persistenceId: String = "simple-accountant" // best practice: make it unique  
  
    // The "normal" receive method       
	override def receiveCommand: Receive = { 
	  // persist one event
      case Invoice(recipient, date, amount) =>         
	    log.info(s"Receive invoice for amount: $amount")  
        persist(InvoiceRecorded(latestInvoiceId, recipient, date, amount)){ e => 
          // SAFE to access mutable state here         
		  latestInvoiceId += 1  
          totalAmount += amount  
          sender() ! "PersistenceACK" // identify the sender of the COMMAND  
          log.info(s"Persisted $e as invoice #${e.id}, for total amount $totalAmount")  
        }  
  
      // persist multiple events
      case InvoiceBulk(invoices) =>      
		val invoiceIds = latestInvoiceId to (latestInvoiceId + invoices.size)  
        val events = invoices.zip(invoiceIds).map { pair =>  
           val id = pair._2  
           val invoice = pair._1  
  
           InvoiceRecorded(id, invoice.recipient, invoice.date, invoice.amount) 
        }  
        persistAll(events) { e =>  
          latestInvoiceId += 1  
          totalAmount += e.amount  
          log.info(s"Persisted SINGLE $e as invoice #${e.id}, for total amount $totalAmount")  
        }  
  
      case Shutdown =>  
        context.stop(self)  
  
      // act like a normal actor  
      case "print" =>  
        log.info(s"Latest invoice id: $latestInvoiceId, total amount: $totalAmount")  
    }  
  
      // Handler that will be called on recovery     
	  override def receiveRecover: Receive = {  
      // best practice: follow the logic in the persist steps of receiveCommand 
		case InvoiceRecorded(id, _, _, amount) =>  
          latestInvoiceId = id  
          totalAmount += amount  
          log.info(s"Recovered invoice #$id for amount $amount, total amount: $totalAmount")  
    }  
  
    /*  
      This method is called if persisting failed.      
	  The actor will be STOPPED.  
      Best practice: start the actor again after a while.      
	  (use Backoff supervisor)     
    */    
	  override def onPersistFailure(cause: Throwable, event: Any, seqNr: Long): Unit = {  
      log.error(s"Fail to persist $event because of $cause")  
      super.onPersistFailure(cause, event, seqNr)  
    }  
  
    /*  
      Called if the JOURNAL fails to persist the event      
	  The actor is RESUMED.     
	*/    
	  override def onPersistRejected(cause: Throwable, event: Any, seqNr: Long): Unit = {  
      log.error(s"Persist rejected for $event because of $cause")  
      super.onPersistRejected(cause, event, seqNr)  
    }  
  }  
  
  val system = ActorSystem("PersistentActors")  
  val accountant = system.actorOf(Props[Accountant], "simpleAccountant")  
  
  for (i <- 1 to 10) {  
    accountant ! Invoice("The Sofa Company", new Date, i * 1000)  
  }  
  
  /*  
    Persistence failures   
  */  
	
  /**  
    * Persisting multiple events    
	*    
	* persistAll    
  */  
	val newInvoices = for (i <- 1 to 5) yield Invoice("The awesome chairs", new Date, i * 2000)  
  //  accountant ! InvoiceBulk(newInvoices.toList)  
  
  // NEVER EVER CALL PERSIST OR PERSISTALL FROM FUTURES.   
	
  /**  
    * Shutdown of persistent actors
	* Best practice: define your own "shutdown" messages    
	*/
	
  //  accountant ! PoisonPill  
  accountant ! Shutdown  
}
```

#### Command Sourcing
Instead of persisting an event, you can persist the command directly.

```Scala
case class Vote(citizenPID: String, candidate: String)

class VotingStation extends PersistentActor with ActorLogging {   
  val citizens: mutable.Set[String] = new mutable.HashSet[String]()  
  val poll: mutable.Map[String, Int] = new mutable.HashMap[String, Int]()  
  
  override def persistenceId: String = "simple-voting-station"  
  
  override def receiveCommand: Receive = {  
    case vote @ Vote(citizenPID, candidate) =>  
      if (!citizens.contains(vote.citizenPID)) {  
          persist(vote) { _ => // COMMAND sourcing  
          log.info(s"Persisted: $vote")  
          handleInternalStateChange(citizenPID, candidate)  
        }  
      } else {  
        log.warning(s"Citizen $citizenPID is trying to vote multiple times!")  
      }  
    case "print" =>  
      log.info(s"Current state: \nCitizens: $citizens\npolls: $poll")  
  }  
  
  def handleInternalStateChange(citizenPID: String, candidate: String): Unit = { 
      citizens.add(citizenPID)  
      val votes = poll.getOrElse(candidate, 0)  
      poll.put(candidate, votes + 1)  
  }  
  
  override def receiveRecover: Receive = {  
    case vote @ Vote(citizenPID, candidate) =>  
      log.info(s"Recovered: $vote")  
      handleInternalStateChange(citizenPID, candidate)  
  }  
}
```

### Multiple Persisting
`persistAll` was concerned with sending multiple events of the same type to the same journal.

You can also persist many different events to the same journal in the handler of one command.

Even though calls to `persist` are asynchronous, the message ordering is guaranteed because persistence is also based on passing messages. Journals are implemented using actors. 

Internally, the journal actor receives a message `WriteMessages(journalBatch, self, instanceId)`. You can imagine calls to `persist` as `journal ! batchedMessages`.

The reaction of the journal is also implemented as a message. As such, the handlers are also called in order.

You can nest `persist` calls inside the callbacks of enclosing persists. They are executed after their enclosing persists. 

```Scala
/*  
  Diligent accountant: with every invoice, will persist TWO events  
  - a tax record for the fiscal authority  
  - an invoice record for personal logs or some auditing authority 
*/  

// COMMAND  
case class Invoice(recipient: String, date: Date, amount: Int)  
  
// EVENTS  
case class TaxRecord(taxId: String, recordId: Int, date: Date, totalAmount: Int)  
case class InvoiceRecord(invoiceRecordId: Int, recipient: String, date: Date, amount: Int)  
  
object DiligentAccountant {  
  def props(taxId: String, taxAuthority: ActorRef) = Props(new DiligentAccountant(taxId, taxAuthority))  
}  
  
class DiligentAccountant(taxId: String, taxAuthority: ActorRef) extends PersistentActor with ActorLogging {  
  
  var latestTaxRecordId = 0  
  var latestInvoiceRecordId = 0  
  
  override def persistenceId: String = "diligent-accountant"  
  
  override def receiveCommand: Receive = {  
    case Invoice(recipient, date, amount) =>  
      // analogous to journal ! TaxRecord  
      persist(TaxRecord(taxId, latestTaxRecordId, date, amount / 3)) { record =>  
        taxAuthority ! record  
        latestTaxRecordId += 1  
  
		// nested persist call
        persist("I hereby declare this tax record to be true and complete.") { declaration =>  
          taxAuthority ! declaration  
        }  
      }  
      // analogous to journal ! InvoiceRecord  
      persist(InvoiceRecord(latestInvoiceRecordId, recipient, date, amount)) { invoiceRecord =>  
        taxAuthority ! invoiceRecord  
        latestInvoiceRecordId += 1  
  
		// nested persist call
        persist("I hereby declare this invoice record to be true.") { declaration =>  
          taxAuthority ! declaration  
        }  
      }  
  }  
  
  override def receiveRecover: Receive = {  
    case event => log.info(s"Recovered: $event")  
  }  
}

class TaxAuthority extends Actor with ActorLogging {  
  override def receive: Receive = {  
    case message => log.info(s"Received: $message")  
  }  
}
```

### Snapshots
Long-lived persistent actors may take a long time to recover. The solution is to save checkpoints (snapshots) along the way.

The `saveSnapshot(state)` method lets you persist anything that's serializable (for example, a queue of messages) to a dedicated persistent store.

You recover this snapshot using a special message, `SnapshotOffer(metadata, contents)`, in the `reciveRecover`method. You convert `contents` back into the type that was saved in the snapshot, loop over `contents` and put it back into state.

```Scala
case SnapshotOffer(metadata, contents) =>  
      log.info(s"Recovered snapshot: $metadata")  
      contents.asInstanceOf[mutable.Queue[(String, String)]].foreach(lastMessages.enqueue(_)) 
```

If you're using snapshots, you're tellng Akka not to recover the entire event history but only the event history since the latest snapshot, which greatly speeds up recovery.

Saving a snapshot is an asynchronous operation, analogous to sending a message to the journal. If the snapshot is successful, the actor will receive a `SaveSnapshotSuccess(metadata)` message. If it fails, the actor receives a `SaveSnapshotFailure(metadata, reason)` message.

The pattern is:
- after each persist, optionally save a snapshot
- if you've saved a snaphot, handle the `SnapshotOffer` message in `receiveRecover`
- (optional,but best practice) handle `SaveSnapshotSuccess` and `SaveSnapshotFailure` in `receiveCommand`.

```Scala
// commands  
case class ReceivedMessage(contents: String) // message FROM your contact  
case class SentMessage(contents: String) // message TO your contact  
  
// events  
case class ReceivedMessageRecord(id: Int, contents: String)  
case class SentMessageRecord(id: Int, contents: String)  
  
object Chat {  
  def props(owner: String, contact: String) = Props(new Chat(owner, contact))  
}  
  
class Chat(owner: String, contact: String) extends PersistentActor with ActorLogging {  
  val MAX_MESSAGES = 10  
  
  var commandsWithoutCheckpoint = 0  
  var currentMessageId = 0  
  val lastMessages = new mutable.Queue[(String, String)]()  
  
  override def persistenceId: String = s"$owner-$contact-chat"  
  
  override def receiveCommand: Receive = {  
    case ReceivedMessage(contents) =>  
      persist(ReceivedMessageRecord(currentMessageId, contents)) { e =>  
        log.info(s"Received message: $contents")  
        maybeReplaceMessage(contact, contents)  
        currentMessageId += 1  
        maybeCheckpoint()  
      }  
    case SentMessage(contents) =>  
      persist(SentMessageRecord(currentMessageId, contents)) { e =>  
        log.info(s"Sent message: $contents")  
        maybeReplaceMessage(owner, contents)  
        currentMessageId += 1  
        maybeCheckpoint()  
      }  
    case "print" =>  
      log.info(s"Most recent messages: $lastMessages")  
    // snapshot-related messages  
    case SaveSnapshotSuccess(metadata) =>  
      log.info(s"saving snapshot succeeded: $metadata")  
    case SaveSnapshotFailure(metadata, reason) =>  
      log.warning(s"saving snapshot $metadata failed because of $reason")  
  }  
  
  override def receiveRecover: Receive = {  
    case ReceivedMessageRecord(id, contents) =>  
      log.info(s"Recovered received message $id: $contents")  
      maybeReplaceMessage(contact, contents)  
      currentMessageId = id  
    case SentMessageRecord(id, contents) =>  
      log.info(s"Recovered sent message $id: $contents")  
      maybeReplaceMessage(owner, contents)  
      currentMessageId = id  
    case SnapshotOffer(metadata, contents) =>  
      log.info(s"Recovered snapshot: $metadata")  
      contents.asInstanceOf[mutable.Queue[(String, String)]].foreach(lastMessages.enqueue(_))  
  }  
  
  def maybeReplaceMessage(sender: String, contents: String): Unit = {  
    if (lastMessages.size >= MAX_MESSAGES) {  
      lastMessages.dequeue()  
    }  
    lastMessages.enqueue((sender, contents))  
  }  
  
  def maybeCheckpoint(): Unit = {  
    commandsWithoutCheckpoint += 1  
    if (commandsWithoutCheckpoint >= MAX_MESSAGES) {  
      log.info("Saving checkpoint...")  
      saveSnapshot(lastMessages)  // asynchronous operation  
      commandsWithoutCheckpoint = 0  
    }  
  }  
}
```

### Recovery
All commands sent during recovery are stashed and will be handled only after recovery has completed.

If recovery fails, the `onRecoveryFailure(cause, event)` method is called and the actor is stopped.

You can customize recovery by overriding the `recovery()` method, for example to recover events up to a certain sequence number or from a tmestamp.

Do not persist more events after a customized recovery, especially if it is incomplete. You might corrupt messages in the meantime.

You can even disable recovery completely with `override def recovery: Recovery = Recovery.none` if you want to start your actor fresh.

The `recoveryFinished` method checks if the recovery has completed, but is abit redundent since you cannot receive commands until the recovery has finished.

At the end of recovery, the actor will receive the `RecoveryCompleted` message, which lets you do some additional initialization. 

In stateless actors, you can use `context.become()` in `receiveCommand` like, normal actors. In `receiveRecover`, the last handler will be used, and only after recovery.

```Scala
case class Command(contents: String)  
case class Event(id: Int, contents: String)  
  
  class RecoveryActor extends PersistentActor with ActorLogging {  
  
    override def persistenceId: String = "recovery-actor"  
  
    override def receiveCommand: Receive = online(0)  
  
    def online(latestPersistedEventId: Int): Receive = {  
      case Command(contents) =>  
        persist(Event(latestPersistedEventId, contents)) { event =>  
          log.info(s"Successfully persisted $event, recovery is ${if (this.recoveryFinished) "" else "NOT"} finished.")  
          context.become(online(latestPersistedEventId + 1))  
        }  
    }  
  
    override def receiveRecover: Receive = {  
      case RecoveryCompleted =>  
        // additional initialization  
        log.info("I have finished recovering")  
  
      case Event(id, contents) =>  
//        if (contents.contains("314"))  
//          throw new RuntimeException("I can't take this anymore!")  
        log.info(s"Recovered: $contents, recovery is ${if (this.recoveryFinished) "" else "NOT"} finished.")  
        context.become(online(id + 1))  
        /*  
          this will NOT change the event handler during recovery          AFTER recovery the "normal" handler will be the result of ALL the stacking of context.becomes.         
		  */    
	}  
  
    override def onRecoveryFailure(cause: Throwable, event: Option[Any]): Unit = {  
      log.error("I failed at recovery")  
      super.onRecoveryFailure(cause, event)  
    }  
  
    //    override def recovery: Recovery = Recovery(toSequenceNr = 100)  
    //    override def recovery: Recovery = Recovery(fromSnapshot = SnapshotSelectionCriteria.Latest)    
	//    override def recovery: Recovery = Recovery.none  
  }
```

### persistAsync
`persistAsync` is a special version of `persist` that relaxes some delivery and ordering guarantees for high-throughput use cases. 

For example, an actor may need to persist events to the journal and also send the events to an events aggregator, such as for a real-time metrics dashboard.

Persisting is already asynchronous. In the case of `persist`, messages received during the time gap between `persist` being called and the callback handler being called are stashed. With `persistAsync` messages are processed, not stashed, during this time gap. 

They are both still based on sending messages and so have shared guarantees:
- persist calls happen in order
- persist callbacks are handled in order

`persistAsync` offers no other guarantees since new messages may be handled in the time gap.

`persistAsync` is often used with command sourcing since it can be faster than creating events.

`persist` is less performant but has stronger ordering guarantees.

`persistAsync` is more performant but is bad if you absolutely need the ordering of events, for example if your state depends on it.

```Scala
case class Command(contents: String)  
case class Event(contents: String)  
  
object CriticalStreamProcessor {  
  def props(eventAggregator: ActorRef) = Props(new CriticalStreamProcessor(eventAggregator))  
}  
  
class CriticalStreamProcessor(eventAggregator: ActorRef) extends PersistentActor with ActorLogging {  
  
  override def persistenceId: String = "critical-stream-processor"  
  
  override def receiveCommand: Receive = {  
    case Command(contents) =>  
      eventAggregator ! s"Processing $contents"  
      // mutate  
      persistAsync(Event(contents)) /*   TIME GAP   */ { e =>  
        eventAggregator ! e  
        // mutate  
      }  
  
      // some actual computation  
      val processedContents = contents + "_processed"  
      persistAsync(Event(processedContents)) /*   TIME GAP   */ { e =>  
        eventAggregator ! e  
      }  
  }  
  
  override def receiveRecover: Receive = {  
    case message => log.info(s"Recovered: $message")  
  }  
}  
  
class EventAggregator extends Actor with ActorLogging {  
  override def receive: Receive = {  
    case message => log.info(s"$message")  
  }  
}
```

## Stores and Serialization
Local LevelDB is a file-based key-value store. It offers compaction. When a specific number of records is reached, the journal files are compacted. It's generally not suitable for production, since it is file-based.

You can configure local snapshot stores. This is file-based and can write anything serializable.

### PostgreSQL
To make a persistent actor compatible with PostgreSQL, you need to import the PostgreSQl library and the `akka-persistence-jdbc` plugin to `build.sbt`.

Add the following configuration to `application.conf`.

```
// application.conf

postgresDemo {  
  akka.persistence.journal.plugin = "jdbc-journal"  
  akka.persistence.snapshot-store.plugin = "jdbc-snapshot-store"  
  
  akka-persistence-jdbc {  
    shared-databases {  
      slick {  
        profile = "slick.jdbc.PostgresProfile$"  
        db {  
          numThreads = 10  
          driver = "org.postgresql.Driver"  
          url = "jdbc:postgresql://localhost:5432/rtjvm"  
          user = "docker"  
          password = "docker"  
        }  
      }  
    }  
  }  
  
  jdbc-journal {  
    use-shared-db = "slick"  
  }  
  
  jdbc-snapshot-store {  
    use-shared-db = "slick"  
  }  
}
```

If using a Docker container to host the PostgreSQL database, you will need an initialization script to create and populate the database when the Docker container is spun up.

When messages are written to the database, by default they are serialized using the default Java serialization. This can be configured.

### Cassandra
Cassandra is a distributed database that is highly available and has high throughput. It does introduce eventual consistency and soft deletion via "tombstones".

To use Cassandra with a persistent actor, you need to add the `akka-persistence-cassandra` and `akka-persistence-cassandra-launcher` plugins to `build.sbt`.

Add the following configuration to `application.conf`

```
cassandraDemo {  
  akka.persistence.journal.plugin = "cassandra-journal"  
  akka.persistence.snapshot-store.plugin = "cassandra-snapshot-store"  
  
  // default values  
}
```

Cassandra will automatically create the keyspace and tables for you. You will need some Docker configuration to have Docker pull in the Cassandra image and spin it up.

### Custom Serialization
Serialization means turning in-memory objects into a recognizable format.

Persisted messages and snapshots are in binary form by default as they use Java serializaton.

The default Java serialization is often not ideal due to memory consumption and performance.

Many Akka serializers are out of date, but this can be customized.

When you send a command to an actor:  
  1. the actor calls `persist`  
  2. serializer serializes the event into bytes  
  3. the journal writes the bytes

The `toBinary` and `fromBinary` methods handle the serialization and deserialization.

```Scala
// command  
case class RegisterUser(email: String, name: String)  
// event  
case class UserRegistered(id: Int, email: String, name: String)  
  
// serializer  
class UserRegistrationSerializer extends Serializer {  
  
  val SEPARATOR = "//"  
  override def identifier: Int = 53278  
  
  override def toBinary(o: AnyRef): Array[Byte] = o match {  
    case event @ UserRegistered(id, email, name) =>  
      println(s"Serializing $event")  
      s"[$id$SEPARATOR$email$SEPARATOR$name]".getBytes()  
    case _ =>  
      throw new IllegalArgumentException("only user registration events supported in this serializer")  
  }  
  
  override def fromBinary(bytes: Array[Byte], manifest: Option[Class[_]]): AnyRef = {  
    val string = new String(bytes)  
    val values = string.substring(1, string.length() - 1).split(SEPARATOR)  
    val id = values(0).toInt  
    val email = values(1)  
    val name = values(2)  
  
    val result = UserRegistered(id, email, name)  
    println(s"Deserialized $string to $result")  
  
    result  
  }  
  
  override def includeManifest: Boolean = false  
}

class UserRegistrationActor extends PersistentActor with ActorLogging {  
  override def persistenceId: String = "user-registration"  
  var currentId = 0  
  
  override def receiveCommand: Receive = {  
    case RegisterUser(email, name) =>  
      persist(UserRegistered(currentId, email, name)) { e =>  
        currentId += 1  
        log.info(s"Persisted: $e")  
      }  
  }  
  
  override def receiveRecover: Receive = {  
    case event @ UserRegistered(id, _, _) =>  
      log.info(s"Recovered: $event")  
      currentId = id  
  }  
}
```

In `application.conf`, you need to define serializers and bind them to a specific serializer class and event.

```
customSerializerDemo {  
  akka.persistence.journal.plugin = "cassandra-journal"  
  akka.persistence.snapshot-store.plugin = "cassandra-snapshot-store"  
  
  akka.actor {  
    serializers {  
      java = "akka.serialization.JavaSerializer"  
      rtjvm = "part3_stores_serialization.UserRegistrationSerializer"  
    }  
  
    serialization-bindings {  
      "part3_stores_serialization.UserRegistered" = rtjvm  
      // java serializer is used by default  
    }  
  }  
}
```


## Advanced Akka Persistence Patterns and Practices
### Schema Evolution with Event Adapters
Schema evolution is a big problem in event-sourcing-based applications. As your app evolves over time, you may need to change the event's structure. What do you do with already persisted data? How do you persist the new data with the new schema alongside the old events?

A naive approach would be to handle recovery of both old and new events in `receiveRecover`. But this would bloat the method every time the schema changes.

Instead, you should set up a `ReadEventAdapter` to upcast or convert events persisted in the journal to some other type.

Events will go: journal -> serializer -> read event adapter -> actor.

The `ReadEventAdapter` will need to implement a method: `fromJournal(event: Any, manifest: String): EventSeq`.

```Scala
override def fromJournal(event:Any, manfest: String): EventSeq = event match {
	case OldEvent(oldFields) => EventSeq.single(NewEvent(newFields))
	case other => EventSeq.sngle(other)
}
```

Now `receiveRecover` only needs to handle the latest version of the persisted event.

You need to add event adapter definitons and bindings to `applicaton.conf`. They are tied to a specific journal.

There is also a `WriteEventAdapter` which dooes the opposite conversion via a `toJournal(event: Any)` method. So the actor would think it's persisting one kind of event, but it is being converted before it reaches the journal. A `WriteEventAdapter` would usually be used for backwards compatibility.

### Detaching Models
The domain model is the definitions of the events the actor thinks it persists.

The data model is the definitions of the objects which actually get persisted.

It's a good practice to make these two models independent. This has the side effect of making schema evolution easier since it is done in the adapter only.

```Scala
object DomainModel {  
  case class User(id: String, email: String, name: String)  
  case class Coupon(code: String, promotionAmount: Int)  
  
  // command  
  case class ApplyCoupon(coupon: Coupon, user: User)  
  // event  
  case class CouponApplied(code: String, user: User)  
}  
  
object DataModel {  
  case class WrittenCouponApplied(code: String, userId: String, userEmail: String)  
  case class WrittenCouponAppliedV2(code: String, userId: String, userEmail: String, username: String)  
}

class ModelAdapter extends EventAdapter {  
  import DomainModel._  
  import DataModel._  
  
  override def manifest(event: Any): String = "CMA"  
  
  // journal -> serializer -> fromJournal -> to the actor  
  override def fromJournal(event: Any, manifest: String): EventSeq = event match {  
    case event @ WrittenCouponApplied(code, userId, userEmail) =>  
      println(s"Converting $event to DOMAIN model")  
      EventSeq.single(CouponApplied(code, User(userId, userEmail, "")))  
    case event @ WrittenCouponAppliedV2(code, userId, userEmail, username) =>  
      println(s"Converting $event to DOMAIN model")  
      EventSeq.single(CouponApplied(code, User(userId, userEmail, username)))  
  
    case other =>  
      EventSeq.single(other)  
  }  
  
  // actor -> toJournal -> serializer -> journal  
  override def toJournal(event: Any): Any = event match {  
    case event @ CouponApplied(code, user) =>  
      println(s"Converting $event to DATA model")  
      WrittenCouponAppliedV2(code, user.id, user.email, user.name)  
  }  
}
```

### Persistence Query
Persistent stores are also used for reading data.

Queries supported by Akka Persistence are:
- select persistence IDs
- select events by persistence ID
- select events across persistence IDs, by tags

The use cases for these queries include:
- which persistent actors are active
- recreate older state
- track how we arrived to the current state
- data processing/aggregation on the events in the entire store.

To access the Persistence Query API, you need a read journal.

The read journal can be queried for which actors are available by calling its `persistenceIds()` method.

```Scala
val persistenceIds = readJournal.persistenceIds()
```

It can be queried by a persistence ID and give you back a source of events.

```Scala
val events = readJournal.eventsByPersistenceId("persistence-query-id", 0, Long.MaxValue) 
```

It can get all events in the store, across persistence IDs, that were previously tagged by an event adapter.

```Scala
val events = readJournal.eventsByTag("my-tag", Offset.noOffset)  
```

The object returned from these persistence queries is a source, whcih can be processed like a collection as an Akka Stream.

```Scala
implicit val materializer = ActorMaterializer()(system) 
events.runForeach { event =>  
	// process the event
}  
```

```Scala
val system = ActorSystem("PersistenceQueryDemo", ConfigFactory.load().getConfig("persistenceQuery")) 

// necessary for streams 
implicit val materializer = ActorMaterializer()(system) 
  
// read journal  
val readJournal = PersistenceQuery(system).readJournalFor[CassandraReadJournal](CassandraReadJournal.Identifier)  
  
// give me all persistence IDs  
val persistenceIds = readJournal.currentPersistenceIds()  

persistenceIds.runForeach { persistenceId =>  
	println(s"Found persistence ID: $persistenceId")  
}  
  
class SimplePersistentActor extends PersistentActor with ActorLogging {  
	override def persistenceId: String = "persistence-query-id-1"  
  
    override def receiveCommand: Receive = {  
      case m => persist(m) { _ =>  
        log.info(s"Persisted: $m")  
      }  
    }  
  
    override def receiveRecover: Receive = {  
      case e => log.info(s"Recovered: $e")  
    }  
  }  
  
val simpleActor = system.actorOf(Props[SimplePersistentActor], "simplePersistentActor")  
  
import system.dispatcher  

system.scheduler.scheduleOnce(5 seconds) {  
	val message = "hello a second time"  
	simpleActor ! message  
}  
  
// events by persistence ID  
val events = readJournal.eventsByPersistenceId("persistence-query-id-1", 0, Long.MaxValue) 

events.runForeach { event =>  
	println(s"Read event: $event")  
}  
  
// events by tags  
val genres = Array("pop", "rock", "hip-hop", "jazz", "disco")  
case class Song(artist: String, title: String, genre: String)  
// command  
case class Playlist(songs: List[Song])  
// event  
case class PlaylistPurchased(id: Int, songs: List[Song])  
  
class MusicStoreCheckoutActor extends PersistentActor with ActorLogging {  
	override def persistenceId: String = "music-store-checkout"  
  
    var latestPlaylistId = 0  
  
    override def receiveCommand: Receive = {  
      case Playlist(songs) =>  
        persist(PlaylistPurchased(latestPlaylistId, songs)) { _ =>  
          log.info(s"User purchased: $songs")  
          latestPlaylistId += 1  
        }  
    }  
  
    override def receiveRecover: Receive = {  
      case event @ PlaylistPurchased(id, _) =>  
        log.info(s"Recovered: $event")  
        latestPlaylistId = id  
    }  
}  
  
class MusicStoreEventAdapter extends WriteEventAdapter {  
    override def manifest(event: Any): String = "musicStore"  
  
    override def toJournal(event: Any): Any = event match {  
      case event @ PlaylistPurchased(_, songs) =>  
        val genres = songs.map(_.genre).toSet  
        Tagged(event, genres)  
      case event => event  
    }  
}  
  
	val checkoutActor = system.actorOf(Props[MusicStoreCheckoutActor], "musicStoreActor")  
	  
	val r = new Random  
	for (_ <- 1 to 10) {  
	val maxSongs = r.nextInt(5)  
	val songs = for (i <- 1 to maxSongs) yield {  
	  val randomGenre = genres(r.nextInt(5))  
	  Song(s"Artist $i", s"My Love Song $i", randomGenre)  
	}  
	
	checkoutActor ! Playlist(songs.toList)  
}  
  
val rockPlaylists = readJournal.eventsByTag("rock", Offset.noOffset)  
rockPlaylists.runForeach { event =>  
println(s"Found a playlist with a rock song: $event")  
}
```