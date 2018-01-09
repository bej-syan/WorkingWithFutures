# Futures in Play

**Play** follows the event-driven web-server architecture, so its default configuration is optimized to work with a 
small number of threads. 

So what we needs to know to get good performance from a Play application is that 
- how to write async actions
- how to adjust Play's thread pool configuration to meet our needs. 

The `Actor.async` builder expects to be given a function of type `Request => Future[Result]`. Actions declared in this 
fashion are not much different from plain `Action { request => ... }` calls, the only difference is that Play knows that
`Action.async` actions are already asynchronous, so it doesn't wrap their contents in a future block. 

##### Blocking and non-blocking controller actions

The `Action.async` builder is useful when implementing actions that perform blocking I/O or CPU-intensive operations 
that take a long time to execute. 

By contrast, the normal `Action` builder doesn't expect an underlying future, but Play will run the body of a normal 
action against the default web worker pool, assuming that it's non-blocking. 

    def echoPath = Action { request =>
      Ok(s"This action has the URI ${request.path}")
    }

The following action, however, is problematic, given its use of the blocking `java.io.File` API:

    def listFiles = Action { implicit request =>
      val files = new java.io.File(".").listFiles
      Ok(files.map(_.getName).mkString(", "))
    }

**Realizing when code is blocking is one of the most important aspects of writing reactive web applications.**

##### Resilient Async Actions
Because futures have a built-in mechanism for failure recovery, it's only natural to apply it to async actions.

##### Custom error handlers

Play has a default error-handling mechanism that can be customized, such as by extending the `DefaultHttpErrorHandler`.
In Some cases, however, it may be useful to configure custom handlers, such as when you're building a REST API. In this
situation it's useful to centralize the error handling in one method

    // Custom error handler attached to a set of futures
    def authenticationErrorHandler: PartialFunction[Throwable, Result] = {
      case UserNotFoundException(userId) =>
        NotFound(
          Json.obj("error" -> s"User with ID $userId was not found")
        )
      case UserDisabledException(userId) =>
        Unauthorized(
          Json.obj("error" -> s"User with Id $userId id disabled")
        )
      case ce: ConnectionException => 
        ServiceUnavailable(
          Json.obj("error" -> "Authentication backed broken")
        )
    }

    val authentication: Future[Result] = ??? 

    // Plugs the recovery handler into the future using the recover method.
    val recoveredAuthentication: Future[Result] = 
      authentication.recover(authenticationErrorHandler)

You define one common recovery handler that knows how to deal with different types of exceptions that may arise when 
invoking an authentication service. 

**Encapsulating this recovery mechanism in a partial function allows it to be reused.**

#### Properly handling timeouts

When working with third-party services, it's a good idea to cap the maximum time a request can take, and to fall back to
another behavior instead of keeping the user waiting for a long time (2 minutes by default in Play). 

    // Handling timeouts
    import play.api.libs.concurrent.Promise
    import scala.concurrent.duration._
    
    case class AuthenticationResult(success: Boolean, error: String)
    
    def authenticate(username: String, password: String) = Action.async {
      implicit request => 
        val authentication: Future[AuthenticationResult] = 
          authenticationService.authenticate(username, password)
        val timeoutFuture = Promise.timeout(
          "Authentication service unresponsive", 2.seconds
        )
        
        Future.firstCompletedOf(
          Seq(authentication, timeoutFuture)
        ).map {
          case AuthenticationResult(success, _) if success => 
            Ok("You can pass")
          case AuthenticationResult(success, error) if !success => 
            Unauthorized(s"You shall not pass: $error")
          case timeoutReason: String => 
            ServiceUnavailable(timeoutReason)
        }
    }


#### Correctly configuring and using execution contexts.

Play's default execution context is backed by an Akka dispatcher and is configured by Play itself. 

Because Play follows the evented server model, the number of hot threads available on the default execution context is
relatively limited. **By default, the dispatcher is set up to create one thread per CPU core, with a maximum of 24 hot
threads in the pool**. 

    // The following is extract from Play's reference configuration:

    akka {
      actor {
        default-dispatcher {
          fork-join-executor {
            parallelism-factor = 1.0
            parallelism-max = 24
            task-peeking-mode = LIFO
          }
        }
      }
    }

For your reactive application to perform well under load, it's important to ensure that your application is entirely
async, or if that's not possible, to adopt a different strategy for dealing with blocking operations. 

#### Specialized execution contexts

It's common to have an application that's mostly asynchronous, except for a few expensive CPU operations or calls to
synchronous libraries. 

If you can identify the special cases that require blocking access, a good approach is to configure a few capped 
execution contexts, and use them in those places. 

For example, let's say your application makes use of a graph database to generate a specialized kind of report, and it
uses a third-party service for resizing images. Those libraries might be performing blocking I/O operations, so it may
be a good idea **to isolate their impact on the default pool** so they don't affect the performance of the application.

###### The first thing you need to do is to configure those contexts accordingly in `conf/application.conf`

    contexts {
      graph-db {
        thread-pool-executor {
          fixed-pool-size = 2
        }
      }
      image-resizer {
        thread-pool-executor {
          core-pool-size-factor = 10.0          // the number of threads will be a multiple of the number of cores and 
                                                // this factor
          max-pool-size-max = 50
        }
      }
    }

###### Next, you need to materialize those execution contexts in your application, such as in a Contexts object.

    object Contexts {
      val graphDB: ExecutionContext = 
        Akka.system.dispatchers.lookup("contexts.graph-db")
      val imageResizer: ExecutionContext = 
        Akka.system.dispatchers.lookup("contexts.image-resizer")
    }

###### Finally, you can use the context in the places where they were designed to be used. 

    /**
     * Wraps the otherwise synchronous code in a future so it will be run on a different execution context.
     */
    def complexReport: Future[Report] = Future { 
      val reportData = queryGraphDb()
      makeReport(reportData)
    }(Contexts.graphDB)

> It's always good to keep the number of threads as low as possible to reduce the amount of context switching and to 
save some memory. It's also good to consider what happens when the pool is exhausted. 

###### **Capping execution context sizes**
- Keep in mind that your aim is to protect the overall application from resource exhaustion
- Consider the consequences of exhaustion for that specific context
- Know the hardware you're running on, and especially how many cores you have at your disposal
- Be aware of the maximum time that tasks running on this context may reuire


###### Bulkheading based on business functions

Depending on the nature of your application, you may take a different approach to organizing your execution contexts and
use the `bulkhead pattern`.  In this approach, instead of dedicating specialized contexts to technical aspects (database
, special third-party services, and so on), you set up contexts based on the functionality of your application. 

Each module uses its own dedicated context across all the technical stack, including blocking database calls or blocking
third-party calls. 


### Testing futures

We can test futures with the `specs2` library (https://etorreborre.github.io/specs2/). 

- `Synchronous` services are mainly tested for the **correctness** of their behavior - whether they behave as expected for a
certain set of inputs. 
- `Asynchronous` services also need to be tested for **timeliness**. This behavior can, in turn, be influenced by the 
timeliness of external dependencies. 
- A third behavior to test is how services respond to delays in those dependencies. 

**To make it easy to test futures, you should make the execution context configurable.**

    trait AuthenticationService {
      def authenticateUser(email: String, password: String)(implicit ec: ExecutionContext): Future[AuthenticationResult]
    }

When using `specs2`'s support for futures, a single-threaded executor is used for the tests, available by default in the
tests.  It's possible to override this configuration or to pass in a specified executor depending on the test case. 

**Asynchronous services require testing for more types of behavior than whether they do the right thing. You also need 
to make sure they do the right thing at the right time.**

    // Testing futures with specs2 

    class AuthenticationServiceSpec extends Specification {
      
      "The AuthenticationService" should {
        val service = new DefaultAuthenticationService
        
        "correctly authenticate Bob Marley" in {
          implicit ee: ExecutionEnv => 
            service.authenticateUser("bob@marley.org", "secret")
            must beEqualTo (AuthenticationSuccessful).await(1, 200.millis)
        }
        
        "not authenticate Ziggy Marley" in { 
          implicit ee: ExecutionEnv => 
            service.authenticateUser("ziggy@marley.org", "secret")
            must beEqualTo (AuthenticationUnsuccessful).await(1, 200.millis)
        }
        
        "fail if it takes too long" in {
          implicit ee: ExecutionEnv => 
            service.authenticateUser("jimmy@hendrix.com", "secret")
            must throwA[RuntimeExecution].await(1, 600.millis)
      }
    }

