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