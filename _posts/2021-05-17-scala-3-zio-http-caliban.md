---
layout: post
title: Trying Scala 3 with zio-http and caliban
---

I decided to start this blog after having fun putting together a little toy project when I wanted to try out some of the new Scala 3 features, since it was finally released 3.0.0 FINAL, a few days ago.

Having recently been using a fair bit of ZIO in projects and also working on a GraphQL api using Sangria, I decided it would be fun to have a go at using [Caliban](https://ghostdogpr.github.io/caliban/), a GraphQL library making use of ZIO.

So to start off, we'll generate a hello world Scala 3 project using:
    
    sbt new scala/scala3.g8

And in the `build.sbt` file, change a few values and import the Caliban and ZIO libraries:

    libraryDependencies ++= Seq(
        "com.github.ghostdogpr" %% "caliban" % "0.10.0",
        "dev.zio" %% "zio" % "1.0.7"
    )

I immediately ran into a problem in that my Scala 3 project had been generated with `val scalaVersion := "3.0.0"` and since this was the day that v3.0.0 was released, obviously the guys at Caliban and ZIO hadn't quite released builds for that Scala version. 
However, they did have builds published for the last release candidate, so the simplest thing for the time being is to set the project scalaVersion to `3.0.0-RC3`.

Caliban has a number of different compatibility libraries for several well known Scala frameworks and http libraries listed very visibly on the site. Already being familiar with Akka-http and Play I thought it would be more interesting to try something else. Initially I was going to try Finatra, but then my eye fell on this:

    "com.github.ghostdogpr" %% "caliban-zio-http" % "0.10.0"

[ZIO-http](https://github.com/dream11/zio-http), an http library based on ZIO! I check out the repo and its had a few releases and a bunch of contributors, so definitely worth looking into!

So with that and also Circe for Json both having builds for 3.0.0-RC3, I add them to my dependencies and now I can start looking at coding.

I find a great medium [article](https://medium.com/@ghostdogpr/graphql-in-scala-with-caliban-part-1-8ceb6099c3c2) on Caliban, written by the developer Pierre Ricadat, with some simple getting started steps. And with a small thematic change, I start implementing my models and some queries and mutations for my API.

So let's start with a simple model, a cat that we can keep a collection of: 
    
```scala
case class Cat(name: String, nicknames: List[String])
```

and to get a list of cats we'll need a query to list them, and also a mutation to add a cat.

In caliban, we define queries (and mutations) as fields on case classes. So let's create a `Queries` class:

```scala
case class Queries(
    // listingCats doesn't require any Env, and shouldn't really fail, so UIO will do
    listCats: UIO[List[Cat]]
)
```
Simple enough, but to add a cat we'll also need an argument, but we can define another case class for that, called `AddCatArgs`, in this case only needing one arg: the cat.

```scala
case class AddCatArgs(cat: Cat)
case class Mutations(
    //let's assume this doesn't need any Env either, and also can't fail, and we don't want anything back from it
    addCat: AddCatArgs => UIO[Unit]
)
```

So there we have a query and a mutation, defined nicely as case classes. When an argument is needed the mutation (or query) is defined as a function.

Now we've described our queries, we need to create a service to actually handle the queries. We can start with a trait:

```scala
trait CatService {
    def listCats: UIO[List[Cat]]
    def addCat(cat: Cat): Task[Cat]
}
```
And a simple implementation, storing cats in a list.
```scala
class CatServiceImpl extends CatService {
    private var data: List[Cat] = List(
    Cat("Faustus", List("Fluffy", "The Baws")),
    Cat("Mephisopheles", List("Smudge", "Mefi")),
    Cat("Dave", List("The great horned one", "Lil' Dave")),
    Cat("Licksworth", List("Sir")))

    def listCats: UIO[List[Cat]] = UIO(data)
    def addCat(cat: Cat): UIO[Unit] = UIO{
    data = data :+ cat
}
```
And so now we can implement our query and mutation, and instances of the `Queries` and `Mutations` are called 'resolvers' in the graphQL nomenclature. 

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

    // Http.collect allows us to define routes using pattern matching.
    // This is perfect for body-less get requests, althoug for POSTs or
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

But now there are compilation errors because Caliban doesn't know how to handle the URL