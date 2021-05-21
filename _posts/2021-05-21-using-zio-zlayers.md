---
layout: post
title: Dependency Injection with ZIO Layers
---

Improving our CatsApp using ZIO Environment types and ZLayers for dependency Injection

So last time we set up our small Caliban, ZIO-Http GraphQL app, with 2 simple endpoints. One to print out the GraphQL Schema, and the other to accept GraphQL requests in the form of HTTP POST requests.

To handle our requests, we implemented a very simple service, that kept an in-memory list of cats which we could query, and edit. Obviously this implementation isn't great if we want our app to properly store data, any changes to the default state would be lost every time the app is restarted. 

However, this would make a suitable implementation for unit testing our application. 

When we want to be able to provide different implementations of a service for testing vs production, dependency injection should probably be the first tool we reach for, and ZIO has a great built in mechanism for this.

### The ZIO Service Pattern

The main abstraction in the ZIO library is the `ZIO` type:

```scala
  //from the ZIO source code
  sealed trait ZIO[-R, +E, +A] ...
```
Where the `A` type is what we want to get back, the `E` type is a possible failure type (such as a Scala Throwable) and the `R` type is an Environment type, something that this effect requires in order to be executed. 

Our service already returns ZIO effects, in the form of `Tasks` and `UIOs`, these are 2 type aliases for ZIO where the `R` type is Any (or 'no environment needed');

In order to define a service which can be injected as an `R` type, we'll follow the `ZIO Service Pattern` which has 4 main parts:
  - A trait `OurService` defining our functions that return effects
  - A type alias for a Has[OurService]
  - An implementation of `OurService` in a ZLayer which can be provided to code that requires a `Has[OurService]` in the `R` type.
  - And finally some functions to be able to create the effects with the `R` type requirement.

So if we take our CatService trait from before:
```scala
trait CatService:
  def listCats: UIO[List[Cat]]
  def addCat(cat: Cat): UIO[Unit]
```

well we can actually reuse this, as it is exactly what part 1 of the pattern is, but we'll put it in an object and rename it. Why will become clearer later.

We'll also add our type alias.

```scala
import models.Cat
import zio._
import java.net.URL

object CatsStore:
 
  type CatsStoreService = Has[CatsStore.Service]

  trait Service:
    def listCats: UIO[List[Cat]]
    def addCat(cat: Cat): UIO[Unit]

```

So now externally to this, if we have a function that requires our service, we would be able to define it as:

```scala
def myFunction: ZIO[CatsStoreService, Throwable, Something] = ???
```
which is a bit tidier than `ZIO[Has[CatsStore.Service], Throwable, Something]`

### ZLayers

So now we need an implementation of our `Service` trait, and we've actually got something just fine, our `CatServiceImpl` with the in-memory list. As mentioned, this would be good for a test implementation.

But we want to provide this implementation as a ZLayer, so it can be provided later before any effects requiring it as an `R` are run.

```scala
val test: ZLayer[Any, Nothing, Has[CatsStore.Service]] = ZLayer.succeed(new Service {
    private var data: List[Cat] = List()

    def listCats: UIO[List[Cat]] = UIO(data)

    def addCat(cat: Cat): UIO[Unit] = UIO{
        data = data :+ cat
    }
  })
```

As we can see from the type definition, ZLayer looks quite similar to ZIO. 
```scala
ZLayer[-RIn, +E, +ROut]
```
It also has 3 types:
 - `RIn` is akin to the `R` in ZIO in that it is a dependency of the ZLayer, something the ZLayer requires to be built. 
 - `E` again is a possible error type,
 - and `ROut` is the type produced that can be provided to build either other layers, or fulfil an effect's environment type.

We need no dependency to build our service and it won't fail during creation, so `RIn` is simply `Any` (no dependency) and `E` is `Nothing` (no error).

Just as ZIO has aliases like `Task`, `RIO` and `UIO` for when there are no dependencies or errors, as does ZLayer. 

So where `UIO[A]` is an alias for `ZIO[Any, Nothing, A]` (no dependency, no error),
`ULayer[ROut]` is an alias for `ZLayer[Any, Nothing, ROut]`, and we'll use that for simplicity.

So now we have our trait, type alias and an implementation (we could also now define a second implementation, for example one that connects to a database), but to complete the pattern, we'll add some functions to make our effects with our service as a dependency:

```scala

import models.Cat
import zio._
import java.net.URL

object CatsStore:
  //Type alias
  type CatsStoreService = Has[CatsStore.Service]
  //Service trait
  trait Service:
    def listCats: UIO[List[Cat]]
    def addCat(cat: Cat): UIO[Unit]

  //Implementation
  val test: ULayer[Has[CatsStore.Service]] = ???

  //Summoner functions
  def listCats: URIO[CatsStoreService, List[Cat]] = ZIO.accessM(_.get.listCats)
  def addCat(cat: Cat): URIO[CatsStoreService, Unit] = ZIO.accessM(_.get.addCat(cat))

```

The 'summoner' functions return an effect that, when provided with our service as `R`, will be able to run our Service functions.

The nice thing about Summoner functions defined in this object along with rest, is that we can now call these functions with `CatStore.listCats`, which reads nicely and gives us what we want.

### Connecting our ZLayer to the rest of the app

Now we can look back to our Caliban code, and see what we need to do to update our schema to use our new service.

```scala
case class Queries(
    //Redefine the type to require our new service as a dependency
    listCats: URIO[CatsStoreService, List[Cat]]
)

case class Mutations(
    addCat: AddCatArgs => URIO[CatsStoreService, Unit]
)
```
And where we define our resolvers:

```scala
// So Caliban can derive our schema, now that our effects require
// a resource type, we need to provide an implementation of 
// GenericSchema and import the 'givens` from within
object schema extends GenericSchema[CatsStoreService]
import schema.{given, *}

// Here instead of needing an instance of a service, we just call
// our summoner functions
val queries = Queries(CatsStore.listCats)
val mutations = Mutations(args => CatsStore.addCat(args.cat))

// And our api now has our Service as a dependency
val api: GraphQL[CatsStoreService] = graphQL(RootResolver(queries, mutations))
```

You'll notice a new Scala 3 feature here, explicitly importing `given`s from our schema. This is now required and a simple `import schema._` will cause compiler errors. 

This is part of the implicits overhaul in Scala 3 designed to make it a bit less 'magic', so we can see here that some `given` are being imported and are needed.

We're almost there, now we need to adjust some types for our HTTP code, and inject our implementation of CatsStore:

```scala
// Just as with the Caliban GraphQL now has a resource type, our
// Our HttpApp also needs one, since executing the request now
// returns an effect requiring a resource
val app: HttpApp[CatsStoreService, Nothing] = Http.collectM[Request] {
    case Method.GET -> Root / "schema" => UIO(Response.text(api.render))
    case r: Request if r.matches(Method.POST -> Root / "graphql") => 
      r.data.asGraphQLRequest
        .flatMap(req => executeRequest(req))
        .catchAll(err => UIO(err.toResponse))
  }

private val PORT = 8090

override def run(args: List[String]): URIO[zio.ZEnv, ExitCode] = {
  Server
    .start(PORT, app)
    // And finally we inject the implemenation of our service
    .provideLayer(CatsStore.test)
    .exitCode
}
```

### Wrapping up

So now we've added some simple dependency injection, and passed in the implementation of our service right where our code is actually run. 

We've seen how to construct the ZIO Service pattern and also how the ZIO-based services of Caliban and ZIO-HTTP work easily with the ZLayer.

For the next improvement we could investigate building another implementation, with a database behind it. In order to do this we might:
 - Define a ZIO Service for the database layer
 - Define our new 'live' implementation of CatsStore as a ZLayer that requires our new Database Layer as an `RIn` type.
 

 The code for this can be found at: https://github.com/ocoulson/zio-http-caliban/tree/using-zlayers