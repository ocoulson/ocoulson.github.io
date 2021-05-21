---
layout: post
title: Trying Scala 3 with zio-http and Caliban
---

I decided to write this post after having fun putting together a little toy project when I wanted to try out some of the new Scala 3 features, since it was finally released 3.0.0 FINAL, a few days ago.

Having recently been using a fair bit of ZIO in projects and also working on a GraphQL api using Sangria, I decided it would be fun to have a go at using [Caliban](https://ghostdogpr.github.io/caliban/), a GraphQL library making use of ZIO.

First we'll generate a hello world Scala 3 project using:
    
    sbt new scala/scala3.g8

Import the Caliban and ZIO libraries:

```scala
libraryDependencies ++= Seq(
    "com.github.ghostdogpr" %% "caliban" % "0.10.0",
    "dev.zio" %% "zio" % "1.0.7"
)
```
I immediately ran into a problem in that my Scala 3 project had been generated with `val scalaVersion := "3.0.0"` and since this was the day that v3.0.0 was released, obviously the guys at Caliban and ZIO hadn't quite released builds for that Scala version. 
However, they did have builds published for the last release candidate, so the simplest thing for the time being is to set the project scalaVersion to `3.0.0-RC3`.

Caliban has a number of different compatibility libraries for several well known Scala frameworks and http libraries listed very visibly on the site. Already being familiar with Akka-http and Play I thought it would be more interesting to try something else. Initially I was going to try Finatra, but then my eye fell on this:

```scala
"com.github.ghostdogpr" %% "caliban-zio-http" % "0.10.0"
```
[ZIO-http](https://github.com/dream11/zio-http), an http library based on ZIO! I check out the repo and its had a few releases and a bunch of contributors, so definitely worth looking into!

So with that and also Circe for Json both having builds for 3.0.0-RC3, I add them to my dependencies and now I can start looking at coding.

I find a great medium [article](https://medium.com/@ghostdogpr/graphql-in-scala-with-caliban-part-1-8ceb6099c3c2) on Caliban, written by the developer Pierre Ricadat, with some simple getting started steps. So with a small thematic change, I start implementing my models and some queries and mutations for my API.

So let's start with a simple model, a Cat that we can keep a collection of: 
    
```scala
case class Cat(name: String, nicknames: List[String])
```

We'll need a query to list cats, and also a mutation to add a new one.

In caliban, we define queries (and mutations) as fields on case classes, and when a query or mutation requires an argument, we also model that as a case class:

```scala
case class Queries(
  // listingCats doesn't require any Env, and shouldn't really fail, so UIO will do
  listCats: UIO[List[Cat]]
)
case class AddCatArgs(cat: Cat)
case class Mutations(
  //let's assume this doesn't need any Env either, and also can't fail, and we don't want anything back from it
  addCat: AddCatArgs => UIO[Unit]
)
```
When an argument is needed the mutation (or query) is defined as a function.

Now we've described our queries, we need to create a service to actually handle the queries. We can start with a trait:

```scala
trait CatService:
  def listCats: UIO[List[Cat]]
  def addCat(cat: Cat): Task[Cat]
```
And a simple implementation, storing cats in a list.
```scala
class CatServiceImpl extends CatService:
  private var data: List[Cat] = List(
    Cat("Faustus", List("Fluffy", "The Baws")),
    Cat("Mephisopheles", List("Smudge", "Mefi")),
    Cat("Dave", List("The great horned one", "Lil' Dave")),
    Cat("Licksworth", List("Sir"))
  )

  def listCats: UIO[List[Cat]] = UIO(data)
  def addCat(cat: Cat): UIO[Unit] = UIO(data = data :+ cat)
```
So now we can implement our query and mutation, and instances of the `Queries` and `Mutations` are called 'resolvers' in the graphQL nomenclature. 

```scala
import caliban.GraphQL._
import caliban.RootResolver


val catService: CatService = ???
val queries: Queries = Queries(
    listCats = catService.listCats
)
val mutations: Mutations = Mutations(
    args => catService.addCat(args.cat)
)
//Create a GraphQL object and pass in our resolvers.
val api = graphQL(RootResolver(queries, mutations))
```
According to the Caliban documentation, calling `api.render` will print out the current graphQL schema, which is great, so I'm thinking I might expose that via an endpoint. Time to look at zio-http.

```scala
import caliban.GraphQL._
import caliban.RootResolver
import zhttp.http._
import zhttp.service._
import zio._

object CatApp extends zio.App {

    val catService: CatService = new CatServiceImpl
    val queries: Queries = Queries(
        listCats = catService.listCats
    )
    val mutations: Mutations = Mutations(
        args => catService.addCat(args.cat)
    )

    val api = graphQL(RootResolver(queries, mutations))

    // Http.collect takes a PartialFunction as an argument,
    // which allows us to define routes using pattern matching.
    // This is perfect for body-less get requests, although for POSTs or
    // similar, we'll need something more

    val app: HttpApp[Any, Nothing] = Http.collect[Request] {
        case Method.Get -> Root / "schema" => Response.text(api.render)
    }

    //Define a port and start our Server in the main 'run' method 
    private val PORT = 8090
    override def run(args: List[String]): URIO[zio.ZEnv, ExitCode] = {
        Server.start(PORT, app).exitCode
}
```

So now a simple `sbt run` and hit `http://localhost:8090/schema` and we get our GraphQL schema printed out. Caliban has autogenerated a schema from our case classes including both a type and an input for Cat as it is returned from the listCats query and used as an argument in the addCat mutation.

```graphql
schema {
  query: Queries
  mutation: Mutations
}

scalar Unit

input CatInput {
  name: String!
  nicknames: [String!]!
}

type Cat {
  name: String!
  nicknames: [String!]!
}

type Mutations {
  addCat(cat: CatInput!): Unit!
}

type Queries {
  listCats: [Cat!]!
}
```

So far so good, but we've not really added much in the way of new Scala 3 features. So let's add a bit to our Cat model: a URL field and an enumerated Colour value. Of course we'll need to update the list of Cats in our CatService.

```scala
import java.net.URL
case class Cat(name: String, nicknames: List[String], picUrl: Option[URL], colour: Colour)

enum Colour:
  case Tabby extends Colour
  case Calico extends Colour
  case Black extends Colour
  case White extends Colour
  case BlackAndWhite extends Colour
  case Ginger extends Colour
```
Caliban already knows how to make a GraphQL enum from our Scala 3 enum, but now there are compilation errors because Caliban doesn't know how to handle the URL in the schema

We need to help the compiler with how to build a schema for URL and since Cat is used as both an input and a return type in GraphQL, we have to provide help for both:

```scala
// Tell the compiler how to make the return type (a string) from a URL
implicit val urlSchema: Schema[Any, URL] = Schema.stringSchema.contramap(_.toString)
// Tell the compiler how to make a url from a string
implicit val urlArgBuilder: ArgBuilder[URL] = ArgBuilder.string.flatMap(
  url => Try(new URL(url)).fold(_ => Left(ExecutionError(s"Invalid URL $url")), Right(_))
)
```

But hold on, Scala 3 provides a entirely new version of implicits. In the case of these implicit values, which are never referenced by name in the code, we can use the new `given` keyword instead, and drop the name reference.

```scala
given Schema[Any, URL] = Schema.stringSchema.contramap(_.toString)
given ArgBuilder[Url] = ArgBuilder.string.flatmap(
  url => Try(new URL(url)).fold(_ => Left(ExecutionError(s"Invalid URL $url")), Right(_))
)
```

And so now our schema is updated, with our enum and URL modeled as a string:
```graphql
enum Colour {
  Black
  BlackAndWhite
  Calico
  Ginger
  Tabby
  White
}

input CatInput {
  name: String!
  nicknames: [String!]!
  picUrl: String
  colour: Colour!
}

type Cat {
  name: String!
  nicknames: [String!]!
  picUrl: String
  colour: Colour!
}
```

Now to be able to query the graphql api, we need a post endpoint, and to read in a GraphQLRequest model (case class provided by Caliban).
Since we need to access the body of the request, we can't use the same pattern match we did for the GET endpoint.
Instead, we need to pull apart the Request and match on various parts of it.

The request looks like this
```scala
final case class Request(endpoint: Endpoint, data: Request.Data)
```
where `Endpoint` is a type alias for a tuple: `(Method, URL)`

So we can add a new pattern and match for the POST method and a "/graphql" path:

```scala
val app: HttpApp[Any, Nothing] = Http.collect[Request] {
  case Method.Get -> Root / "schema" => Response.text(api.render)
  case r: Request if r.endpoint._1 == Method.POST && r.endpoint._2.path == Root / "graphql" => Response.text("Hello World")
}
```

But this looks a little ugly, so we can use another new Scala 3 feature, `extension methods` to clean it up a bit. These used to be implemented using implicit classes, but now use the `extension` keyword:

```scala
object Extensions:
  extension (r: Request)
    def matches(route: Route): Boolean = r.endpoint._1 == route._1 && r.endpoint._2.path == route._2

```
We've written this also with the new indentation syntax from Scala 3, notice the lack of {}.

So we can import this extension method and clean up our HttpApp:

```scala
val app: HttpApp[Any, Nothing] = Http.collect[Request] {
  case Method.Get -> Root / "schema" => Response.text(api.render)
  case r: Request if r.matches(Method.POST -> Root / "graphql") => Response.text("Hello World")
}
```
This looks cleaner, and we can easily reuse it for further POSTs, PUTs and other HTTP methods if we want. Although, with GraphQL, we may not need any further endpoints.

Now the data from the `Request` can be accessed with simple `r.data`, and we can read in the data, parse it to a GraphQLRequest using Circe and pass it on to our `api` to complete the request.

```scala
//Define another extension in our Extensions object, and capture any failures with a Task
extension (d: Request.Data)
    def asGraphQLRequest: Task[GraphQLRequest] = d.content match {
      case HttpData.CompleteData(data) => 
        ZIO
          .fromEither(Option(data.map(_.toChar).mkString).toRight(new Exception("Error Getting data")))
          .flatMap(jsonString => ZIO.fromEither(io.circe.parser.decode[GraphQLRequest](jsonString)))
        
      case _                           => ZIO.fail(new Exception("Incomplete data"))
    }
```

Now that our function is using a `zio.Task`, we need to change our HttpApp function a little:

```scala
                                //Use CollectM when the PartialFunction returns an effect
val app: HttpApp[Any, Nothing] = Http.collectM[Request] {
  case Method.Get -> Root / "schema" => UIO(Response.text(api.render))
  case r: Request if r.matches(Method.POST -> Root / "graphql") => r.data.asGraphQLRequest ...
}
```

Now that we have a GraphQLRequest (maybe), we need to create an `interpreter` from our api, and run our request through it.
```scala
val api = graphQL(RootResolver(queries, mutations))

val app: HttpApp[Any, Nothing] = Http.collectM[Request] {
  case Method.Get -> Root / "schema" => UIO(Response.text(api.render))
  case r: Request if r.matches(Method.POST -> Root / "graphql") => 
    r.data.asGraphQLRequest
      .flatMap(request => for{
        interpreter <- api.interpreter
        result <- interpreter.executeRequest(request)
      } yield Response.jsonString(result.data.toString))
}
```
Because this call could fail, for example when reading the graphql content from the request body, we should handle the failure case of the returned effect.

```scala
val app: HttpApp[Any, Nothing] = Http.collectM[Request] {
  case Method.Get -> Root / "schema" => UIO(Response.text(api.render))
  case r: Request if r.matches(Method.POST -> Root / "graphql") => 
    r.data.asGraphQLRequest
      .flatMap(request => for{
        interpreter <- api.interpreter
        result <- interpreter.executeRequest(request)
      } yield Response.jsonString(result.data.toString))
      .catchAll(err => UIO(Response.fromHttpError(HttpError.BadRequest(err.getMessage))))
}
```
The '400' Bad Request is fine for problems in the reading of the request, but it would probably be misleading for some errors thrown during request execution. For this example, we'll leave it like this, but with a more fleshed out app, we would want to design in some better error handling, possibly a custom error type for which we can return the correct HTTP status.

It's also worth noting that GraphQL errors are often returned as HTTP 200 responses, but with error messages within, due to the ability to perform multiple queries / mutations in a single request, some may succeed while others fail.

Now the finished app looks like this:

Models:
```scala
import java.net.URL
import zio._

case class Cat(name: String, nicknames: List[String], picUrl: Option[URL], colour: Colour)

enum Colour:
  case Tabby extends Colour
  case Calico extends Colour
  case Black extends Colour
  case White extends Colour
  case BlackAndWhite extends Colour
  case Ginger extends Colour

case class AddCatArgs(cat: Cat)

case class Queries(listCats: UIO[List[Cat]])
case class Mutations(addCat: AddCatArgs => UIO[Unit])
```
Service:
```scala
trait CatService:
  def listCats: UIO[List[Cat]]
  def addCat(cat: Cat): UIO[Unit]

class CatServiceImpl extends CatService:
  private def Url(s: String)= Try(new URL(s)).toOption
  private var data: List[Cat] = List(
      Cat("Faustus", List("Fluffy", "The Baws"), Url("https://i.natgeofe.com/n/3861de2a-04e6-45fd-aec8-02e7809f9d4e/02-cat-training-NationalGeographic_1484324.jpg"), Colour.Ginger),
      Cat("Mephisopheles", List("Smudge", "Mefi"), None, Colour.Black),
      Cat("Dave", List("The great horned one", "Lil' Dave"), Url("https://static.scientificamerican.com/sciam/cache/file/32665E6F-8D90-4567-9769D59E11DB7F26_source.jpg?w=590&h=800&7E4B4CAD-CAE1-4726-93D6A160C2B068B2"), Colour.Ginger),
      Cat("Licksworth", List("Sir"), Url("https://cdn.mos.cms.futurecdn.net/VSy6kJDNq2pSXsCzb6cvYF-1024-80.jpg.webp"), Colour.Tabby )
  )

  def listCats: UIO[List[Cat]] = UIO(data)
  def addCat(cat: Cat): UIO[Unit] = UIO(data = data :+ cat)
```
Main App:
```scala
import zio._
import zhttp.http._
import zhttp.service._
import caliban.GraphQL.graphQL
import caliban.RootResolver
import caliban.schema.{ ArgBuilder, Schema }
import caliban.CalibanError.ExecutionError
import caliban.GraphQLRequest
import io.circe.Json
import io.circe.syntax._

import scala.util.Try
import java.net.URL
import Extensions._

object CatApp extends zio.App {

  given Schema[Any, URL] = Schema.stringSchema.contramap(_.toString)
  given ArgBuilder[URL] = ArgBuilder.string.flatMap(
    url => Try(new URL(url)).fold(_ => Left(ExecutionError(s"Invalid URL $url")), Right(_))
  )

  val catService: CatService = new CatStore

  val queries = Queries(
    catService.listCats,
    args => catService.findCat(args.name),
    catService.randomCatPicture
  )

  val mutations = Mutations(
    args => catService.addCat(args.cat),
    args => catService.editCatPicture(args.name, args.picUrl)
  )
  
  val api = graphQL(RootResolver(queries, mutations))

  def executeRequest(request: GraphQLRequest) = for {
    interpreter <- api.interpreter
    res <- interpreter.executeRequest(request)
  } yield Response.jsonString(res.data.toString)

  val app: HttpApp[Any, Nothing] = Http.collectM[Request] {
    case Method.GET -> Root / "schema" => UIO(Response.text(api.render))
    case r: Request if r.matches(Method.POST -> Root / "graphql") => 
      r.data.asGraphQLRequest
        .flatMap(req => executeRequest(req))
        .catchAll(err => UIO(err.toResponse))
  }

  private val PORT = 8090

  override def run(args: List[String]): URIO[zio.ZEnv, ExitCode] = {
    Server.start(PORT, app).exitCode
  }
}

object Extensions:
  extension (r: Request) 
    def matches(route: Route): Boolean = r.endpoint._1 == route._1 && r.endpoint._2.path == route._2
  
  extension (d: Request.Data)
    def asGraphQLRequest: Task[GraphQLRequest] = d.content match {
      case HttpData.CompleteData(data) => 
        ZIO
          .fromEither(Option(data.map(_.toChar).mkString).toRight(new Exception("Error Getting data")))
          .flatMap(jsonString => ZIO.fromEither(io.circe.parser.decode[GraphQLRequest](jsonString)))
        
      case _                           => ZIO.fail(new Exception("Incomplete data"))
    }

  
  extension (th: Throwable)
    def toResponse: UResponse = Response.fromHttpError(HttpError.BadRequest(th.getMessage))

```

In terms of how to take this further, here are some suggestions which I might try at some point:

### Scala 3: 
- Implement some models which can make use of `opaque type aliases` to provide type safety for fields on a model that have the same type
- A more interesting `enum` structure to make an ADT, see how that would work with the Graphql Schema
- Implement a `type class` using the new `given` and `using` keywords instead of implicits

### Other:
- Make the CatService into a ZIO service and use `ZLayers` and Resource types to inject an instance of the service into our GraphQL API, Caliban being based on ZIO also has the concept of Resource types. We could make our existing CatServiceImpl into a mock or test version and hook up a database for live code.
- Look into how you might implement some security features, such as Authentication and CORS using `zio-http`


The code for this project can be found at: https://github.com/ocoulson/zio-http-caliban