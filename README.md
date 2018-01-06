# Working with **Future**s

**Futures are at the foundation of asynchronous and reactive programming in Scala.**

A `Future` is a monadic data structure, which means that it can easily be composed with other futures.

Futures help address the problem of programming in an asynchronous fashion in several ways:
- They encapsulate the result of asynchronous computations in a composable data structure.
- They transparently handle failure cases, propagating them along chained futures.
- They provide a mechanism for scheduling the execution of asynchronous tasks on a thread pool.

The *Play*'s WS library is asynchronous and returns future results.

A `scala.concurrent.Future[T]` can be thought of as a box that will eventually contain a value of type `T` if it
succeeds. If it fails, the `Throwable` at the origin of the failure will be kept.

As soon as a `Future` is declared, it will start running, which means that the computation it tries to achieve will be
executed asynchronously.

    // For example,
    // you can use Play's WS library to execute a GET request against the Play Framework website:
    val response: Future[WSResponse] =
      WS.url("http://www.playframework.com").get()

    // This call will return immediately and let you continue to do other things.

    import scala.util.{Success, Failure}

    response onComplete {
      case Success(res) => println(s"Success: res.body")
      case Failure(t) => t.printStackTrace()
    }

    // The `onComplete` handler takes a callback of type `Try[T] => U`. 
    // The success and failure of the `Future` are encoded using the `Success` and `Failure` case classs, which are the
    // two possible implementations of `Try`.

**Being able to transform the content of a future without having to wait for it to complete is key to building more
complex asynchronous computation pipelines.**

    val response: Future[WSResponse] = 
      WS.url("http://www.playframework.com").get()

    val siteOnline: Future[Option[Boolean]] = response.map { r => 
      r.status == 200 
    } recover { // Handle recovery with the recover function.
      case ce: java.net.ConnectException => None
    }

    siteOnline.foreach { isOnline => 
      if (isOnline) { 
        println("The Play site is up")
      } else {
        println("The Play site is down")
      }
    }

**One of the nicest features of Scala's futures is that you can compose them.**

    def siteAvailable(url: String): Future[Boolean] = 
      WS.url(url).get().map { r => 
        r.status == 200 
      }

    val playSiteAvailable = 
      siteAvailable("http://www.playframework.com")
    val playGithubAvailable = 
      siteAvailable("http://github.com/playframework")

    val allSitesAvailable: Future[Boolean] = 
      for {
        siteAvailable <- playSiteAvailable
        githubAvailable <- playGithubAvailable
      } yield (siteAvailable && githubAvailable)

    val overallAvailability: Future[Option[Boolean]] = 
      allSitesAvailable.map { a => 
        Option(a)
      } recover {
        case ce: java.net.ConnectException => None
      }

**In order to run, a future needs to have access to an** `ExecutionContext`, **which takes care of running the 
asynchronous tasks.**
**An** `ExecutionContext` **is typically backed by a plain old** `ThreadPool`.
**Scala's concurrent library provides a default global execution context, and Play also has a default execution context
that can be imported as follows**

    // Before Play 2.6
    import play.api.libs.concurrent.Execution.Implicits._

    // Declaring a custom execution context and running a simple future block on it
    import scala.concurrent._
    import java.util.concurrent.Executors

    implicit val ec = ExecutionContext.fromExecutor(
      Executors.newFixedThreadPool(2)
    }

    val sum: Future[Int] = Future { 1 + 1 }
    sum.foreach { s => println(s) }

Although it may be handy to use a default execution context for running futures when you're getting started with a 
project, **a better strategy for avoiding trouble later on is to design your service APIs in such a way that an 
execution context can be passed to them**

Futures should primarily be used when there's a blocking operation happening. **Blocking operations are mainly I/O
bound, such as network calls or disk access**. 

    // Creating a future over a blocking operation
    import scala.concurrent._
    import scala.concurrent.ExecutionContext.Implicits.global
    import java.io.File

    def fileExists(path: String): Future[Boolean] = Future {
      new java.io.File(path).exists
    }

**Note** that this won't magically turn the blocking code into something asynchronous! 

The *java.io.File* API call will still be blocking. But now you can run this code on a different execution context, 
which means that it won't use the threads of your default application's execution context, which is important to keep in
mind, especially when working with Play. 

A **Future Block** doesn't just create a new future; it schedules its execution against an execution context. 

**You shouldn't create futures to wrap purely CPU-bound operations**

**Creating a future is a costly operation because it involves switching the computation to another execution context and
paying the cost of context switching**.

### **Async code doesn't equal faster code**
Async code is non-blocking, which means that it won't monopolize(垄断) threads while waiting for a result. 
There're costs associated with async due to the overhead introduced by context switching. Depending on how often the 
context is switched, this overhead can be more or less important, but it's always there. 

### Telling the execution context about blocking code
There's a `blocking` marker that allows yo to tell the execution context that a certain portion of code is blocking. 
This is useful because the execution context will then be able to respond appropriately. 

    import scala.concurrent._
    import scala.concurrent.ExecutionContext.Implicits.global
    
    import java.io.File
    
    def fileExists(path: String): Future[Boolean] = Future {
      blocking {
        new java.io.File(path).exists
      }
    }

 


