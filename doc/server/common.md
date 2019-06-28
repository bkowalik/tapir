# Common server options

## Status codes

By default, successful responses are returned with the `200 OK` status code, and errors with `400 Bad Request`. However,
this can be customised by specifying how an [output maps to the status code](../endpoint/ios.html#status-codes).

## Defining an endpoint together with the server logic

It's possible to combine an endpoint description with the server logic in a single object,
`ServerEndpoint[I, E, O, S, F]`. Such an endpoint contains not only an endpoint of type `Endpoint[I, E, O, S]`, but
also a logic function `I => F[Either[E, O]]`, for some effect `F`.

For example, the book example can be more concisely written as follows:

```scala
import tapir._
import tapir.server.akkahttp._
import scala.concurrent.Future
import akka.http.scaladsl.server.Route

val countCharactersServerEndpoint: ServerEndpoint[String, Unit, Int, Nothing, Future] =
  endpoint.in(stringBody).out(plainBody[Int]).serverLogic { s =>
    Future.successful(Right[Unit, Int](s.length))
  }

val countCharactersRoute: Route = countCharactersServerEndpoint.toRoute
```

A `ServerEndpoint` can then be converted to a route using `.toRoute`/`.toRoutes` methods (without any additional
parameters), or to documentation.

Moreover, a list of server endpoints can be converted to routes or documentation as well:

```scala
val endpoint1 = endpoint.in("hello").out(stringBody)
  .serverLogic { _ => Future.successful("world") }

val endpoint2 = endpoint.in("ping").out(stringBody)
  .serverLogic { _ => Future.successful("pong") }

val route: Route = List(endpoint1, endpoint2).toRoute
```

Note that when dealing with endpoints which have multiple input parameters, the server logic function is a function
of a *single* argument, which is a tuple; hence you'll need to pattern-match using `case` to extract the parameters:

```scala
val echoEndpoint = endpoint
  .in(query[Int]("count"))
  .in(stringBody)
  .out(stringBody)
  .serverLogic { case (count, body) =>
     Future.successful(body * count)
  }
```

## Server options

Each interpreter accepts an implicit options value, which contains configuration values for:

* how to create a file (when receiving a response that is mapped to a file, or when reading a file-mapped multipart 
  part)
* how to handle decode failures  

To customise the server options, define an implicit value, which will be visible when converting an endpoint or multiple
endpoints to a route/routes. For example, for `AkkaHttpServerOptions`:

```
implicit val customServerOptions: AkkaHttpServerOptions = AkkaHttpServerOptions.default.copy(...) 
```
  
### Handling decode failures

Quite often user input will be malformed and decoding will fail. Should the request be completed with a 
`400 Bad Request` response, or should the request be forwarded to another endpoint? By default, tapir follows OpenAPI 
conventions, that an endpoint is uniquely identified by the method and served path. That's why:

* an "endpoint doesn't match" result is returned if the request method or path doesn't match. The http library should
  attempt to serve this request with the next endpoint.
* otherwise, we assume that this is the correct endpoint to serve the request, but the parameters are somehow 
  malformed. A `400 Bad Request` response is returned if a query parameter, header or body is missing / decoding fails, 
  or if the decoding a path capture fails with an error (but not a "missing" decode result).

This can be customised by providing an implicit instance of `tapir.server.DecodeFailureHandling`, which basing on the 
request, failing input and failure description can decide, whether to return a "no match" or a specific response.

Only the first failure is passed to the `DecodeFailureHandling`. Inputs are decoded in the following order: method, 
path, query, header, body.

## Extracting common route logic

Quite often, especially for [authentication](../endpoint/auth.html), some part of the route logic is shared among 
multiple endpoints. However, these functions don't compose in a straightforward way, as authentication usually operates
on a single input, which is only a part of the whole logic's input. Suppose you have the following methods:

```scala
type AuthToken = String

def authFn(token: AuthToken): Future[Either[ErrorInfo, User]]
def logicFn(user: User, data: String, limit: Int): Future[Either[ErrorInfo, Result]]
```

which you'd like to apply to an endpoint with type:

```scala
val myEndpoint: Endpoint[(AuthToken, String, Int), ErrorInfo, Result, Nothing] = ...
```

To avoid composing these functions by hand, tapir defines helper extension methods, `andThenFirst` and `andTheFirstE`. 
The first one should be used when errors are represented as failed wrapper types (e.g. failed futures), the second
is errors are represented as `Either`s. 

This extension method is defined in the same traits as the route interpreters, both for `Future` (in the akka-http
interpreter) and for an arbitrary monad (in the http4s interpreter), so importing the package is sufficient to use it:

```scala
import tapir.server.akkahttp._
val r: Route = myEndpoint.toRoute((authFn _).andThenFirstE((logicFn _).tupled))
```

Writing down the types, here are the generic signatures when using `andThenFirst` and `andThenFirstE`:

```scala
f1: T => Future[U]
f2: (U, A1, A2, ...) => Future[O]
(f1 _).andThenFirst(f2): (T, A1, A2, ...) => Future[O]

f1: T => Future[Either[E, U]]
f2: (U, A1, A2, ...) => Future[Either[E, O]]
(f1 _).andThenFirstE(f2): (T, A1, A2, ...) => Future[Either[E, O]]
```

## Exception handling

There's no exception handling built into tapir. However, tapir contains a more general error handling mechanism, as the
endpoints can contain dedicated error outputs.

If the logic function, which is passed to the server interpreter, fails (i.e. throws an exception, which results in
a failed `Future` or `IO`/`Task`), this is propagated to the library (akka-http or http4s). 

However, any exceptions can be recovered from and mapped to an error value. For example:

```scala
type ErrorInfo = String

def logic(s: String): Future[Int] = ...

def handleErrors[T](f: Future[T]): Future[Either[ErrorInfo, T]] =
  f.transform {
    case Success(v) => Success(Right(v))
    case Failure(e) =>
      logger.error("Exception when running endpoint logic", e)
      Success(Left(e.getMessage))
  }

endpoint
  .errorOut(plainBody[ErrorInfo])
  .out(plainBody[Int])
  .in(query[String]("name"))
  .toRoute((logic _).andThen(handleErrors))
```

In the above example, errors are represented as `String`s (aliased to `ErrorInfo` for readability). When the
logic completes successfully an `Int` is returned. Any exceptions that are raised are logged, and represented as a
value of type `ErrorInfo`. 

Following the convention, the left side of the `Either[ErrorInfo, T]` represents an error, and the right side success.

Alternatively, errors can be recovered from failed effects and mapped to the error output - provided that the `E` type
in the endpoint description is itself a subclass of exception. This can be done using the `toRouteRecoverErrors` method.