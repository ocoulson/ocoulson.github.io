---
title: Circe - Json processing in Scala
layout: post
---

Working with JSON in Scala using Circe (pronouced 'Ser-see')

---

There are many different options for working with JSON data in Scala. The api provided by Circe I've found to be the most elegant and easy to use (once you get used to it), for the purposes I've had.

The major use I've had for Circe is in the serialisation and deserialisation (or encoding and decoding) of JSON data (strings) into Scala case class models to be used in business logic. 

First we'll look at the way that Circe represents a `Json` object and then we'll look at how we might use it to read and write JSON data in an api.

A `Json` object in Circe is represented with a class hierarchy that resembles this:

```scala
//note this is not the actual code, but a simplified version of it
trait Json
case class JBoolean(value: Boolean) extends Json
case class JNumber(value: JsonNumber) extends Json
case class JString(value: String) extends Json
case class JArray(value: Vector[Json]) extends Json
case class JObject(value: Map[String, Json]) extends Json
case object JNull extends Json
```
So for example:

```
{
  "someObject": {
    "integer": 1,
    "foo": true,
    "bar": null
  }
}
```
would be represented more or less as:
```scala
JsObject(
  Map(
    "someObject" -> JsObject(
      Map(
        "integer" -> JsNumber(1),
        "foo" -> JsBoolean(true),
        "bar" -> JsNull
      )
    )
  )
)
```

When working with JSON data, I most often am dealing with some sort of JSON String, for example something read from the body of an HTTP POST request, or the response to an HTTP GET request. 
So this `Json` representation usually acts as a sort of middle ground between the JSON String and some sort of data model case class. 

Circe provides many options for working with the raw JSON object; reading parts of it, modifying it, but I won't be focussing on that in this article.

---

## Decoding JSON data into a data model

So for our example, we'll have a simple system for stock and orders.

An item has a name, an id and a value (in £), and so will be represented in JSON like this:
```
{
  "id": "C123",
  "name": "cable",
  "value": 1.55
}
```

Lets say we have an api endpoint which will create an item in the system. 

The item will be read in from a POST request and added to a database. So first we need to read in the item JSON, decode it into our Item model and return an error if the JSON doesn't fit the Item schema.

```scala
//HTTP request and responses
case class Request(body: String)

sealed trait Response
case class Ok(message: String) extends Response
case class Error(message: String) extends response
```

Using the `decode` function, we can tell the compiler with a type parameter, what type we expect the decoded JSON to fit. 
```scala
import io.circe.parser.decode
case class Item(id: String, name: String, value: Double)

def addItem(request: Request): Response = {
  val item = decode[Item](request.body)
}
```

However the above code wouldn't compile because circe doesn't know how to make an `Item`, for this we need to provide it with an instance of the `Decoder` typeclass for `Item`. A `Decoder[T]` is basically instructions for how to turn a `Json` into a `T` and can be written like this:

```scala
import io.circe.Decoder
// Circe uses a Cursor to traverse a json string and get various Json elements. Then for each one we transform it into the expected type (String, Double etc.)
val itemDecoder: Decoder[Item] = new Decoder[Item] {
    override def apply(c: HCursor): Result[Item] = for {
      id <- c.downField("id").as[String]
      name <- c.downField("name").as[String]
      value <- c.downField("value").as[Double]
    } yield Item(id, name, value)
  }
```

The decoder function will return a `Result[T]` which is just an alias for `Either[Error, T]` where `Error` is circe's own error type. 2 common types of errors are:
  - ParsingFailure - When a JSON string is not valid JSON
  - DecodingFailure - When a Json object cannot be decoded into the expected model according to the Decoder rules.

Now we have a decoder we can pass it to the `decode` function

```scala
def addItem(request: Request): Response = {
  val item = decode[Item](request.body)(itemDecoder)
}
```

Alternatively, if we define the decoder as an implicit value on the `Item` companion object, the `decode` function can summon the typeclass instance implicitly:

```scala
object Item {
  implicit val decoder: Decoder[Item] = ???
}

def addItem(request: Request): Response = {
  val item = decode[Item](request.body)
}
```

## Encoding a model to a Json

So for our endpoint, now that we have either an Item or an error, we can store it in the database and then return the created item (or an error) to the caller.

```scala

object DBService {
  //For simplicity the call to the db is syncronous but might return a throwable for an error
  def storeItemInDb(item: Item): Either[Throwable, Item] = ???
}

def addItem(request: Request): Response = {
  decode[Item](request.body) match {
    case Left(err) => Error("Input is not a valid Item")
    case Right(item) => DBService.storeItemInDb(item) match {
      case Left(th) => Error(s"Failure storing item - ${th.getMessage}")
      case Right(createdItem) => 
        import io.circe.syntax._
        Ok(createdItem.asJson.noSpaces)
    }
  }
}
```

So above, the line `createdItem.asJson.noSpaces` is first creating a `Json` from the 'createdItem' `Item` and then printing it to a string with no spaces or newlines e.g.: `{"foo":"bar"}`. In order to use the `asJson` extension message we must import `io.circe.syntax._` in the scope.

However, again there is a problem because although Circe knows how to read an `Item` from a `Json`, it doesn't know how to write an `Item` into a `Json`. 

For this we use another typeclass: `Encoder[T]` which is instructions for how to write a `Json` from a `T`, and again we'll define the instance for `Item` as an implicit value on the `Item` companion object:


```scala
import io.circe._
object Item {
  implicit val decoder: Decoder[Item] = ???

  implicit val encoder: Encoder[Item] = new Encoder[Item] {

    override def apply(a: Item): Json =
      Json.obj(
        "id"    -> Json.fromString(a.id),
        "name"  -> Json.fromString(a.name),
        "value" -> Json.fromDoubleOrNull(a.value)
      )
  }
}
```

```scala
import io.circe.syntax._
//Now we can pass the encoder explicitly:
Ok(createdItem.asJson(Item.encoder).noSpaces)

//or let the compiler resolve it implicitly:
Ok(createdItem.asJson.noSpaces)
```


## Automatic and Semi-automatic derivation

So far so good, we've used the `Decoder[T]` and `Encoder[T]` type classes to tell how to serialise and deserialise an `Item` into a `Json` and used this to read and write this from our API.

However, writing out the type class instances for `Decoder[Item]` and `Encoder[Item]` was a little verbose, especially for such a simple case class.

Circe gives us a mechanism to auto generate these encoders and decoders for a given type class using the `io.circe.generic` library.

```scala
//We have not defined the Encoder and Decoder for Item anywhere in scope

import io.circe.parser.decode
import io.circe.syntax._
import io.circe.generic.auto._

object Example {
  val itemString = """{"id":"id1","name":"name1","":1.0}"""
  val item = decode[Item](itemString) //The decoder is generated automatically by the compiler

  val itemJson = item.asJson //The encoder is also generated automatically
}
```

Here we can see that by importing `io.circe.generic.auto._` in scope where the `decode` and `asJson` functions are called, the compiler can then go and create instances for `Encoder[Item]` and `Decoder[Item]`.

This is a very useful tool for quickly and easily working with JSON and models, however it can be a little less easy to reason about as there is a little implicit "magic" going on. Additionally this can add a bit of a compilation overhead.

There is an alternative to the fully automatic derivation: semi-automatic derivation. Here we define our type class instances (probably implicitly on the model companion object), but we allow the compiler to derive the instance. This is a bit more explicit and allows you to reason more about where the codeces are.

```scala
import io.circe.{Decoder, Encoder}
import io.circe.generic.semiauto.{deriveDecoder, deriveEncoder}
object Item {
  val decoder: Decoder[Item] = deriveDecoder[Item]
  val encoder: Encoder[Item] = deriveEncoder[Item]
}
```

As a further simplification, circe also has a type class `Codec[T]` which extends both Decoder and Encoder, which you can also derive automatically:

```scala
import io.circe.Codec
import io.circe.generic.semiauto.deriveCodec
object Item {
  val codec: Codec[Item] = deriveCodec[Item]
}
```

So now anywhere `Item` is in scope, there will be an instance of `Codec[Item]` in the case of wanting to decode / encode them from / to Json.