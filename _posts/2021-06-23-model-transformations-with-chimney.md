---
layout: post
title: Model Transformations with Chimney
---

A great little library for transfomations between similar models.

---

There are several scenarios when working on a project where we might need to transform one model (case class) into another similar model.

On a project I work on, we try to keep our different domain models separate, so for example we'll have duplicated or nearly duplicated models for an entity for when it is stored in a database from when it is returned on the API. 

There are several reasons for doing this, including:
- it keeps your code decoupled from other parts of the application, 
- it can shield the different parts of the application from changes made in another part.

BUT, converting an object into another very similar or identical class can be a bit long winded. 

```scala
case class A(a: String, b: Int, c: Double, x: Boolean, y: String, z: Char)

case class B(a: String, b: Int, c: Double, x: Boolean, y: String, z: Char)

def aToB(a: A): B = B(a.a, a.b, a.c, a.x, a.y, a.z)
```

In a small example like this, it's not too much, but when your model has 15 fields, with nested objects, not just 'primitives', this can get big. 
And let's not forget we probably need to do the transformation the other way too.

Let's consider another example of when you might want to transform similar models

```scala
case class Person(id: PersonId, name: String, dob: Date, address: Address)
case class Address(houseNumber: HouseNumber, streetName: StreetName, postcode: Postcode)

case class PersonId(val value: String) extends AnyVal
case class PostCode(val value: String) extends AnyVal
case class StreetName(val value: String) extends AnyVal
case class HouseNumber(val value: Int) extends AnyVal
```

Say we have an app that keeps track of people, with a REST API POST endpoint to create a person, sending a Json that looks like this:

```json
{
  "name": "Joe",
  "dob": "11/12/1982",
  "address": {
    "houseNumber": 12,
    "streetName": "Meteor Drive",
    "postcode": "NN54 JB93"
  }
}
```

And we've used our favourite Json library to read this into a couple of case classes:

```scala
case class PersonInput(name: String, dob: Date, address: AddressInput)
case class AddressInput(houseNumber: Int, streetName: String, postcode: String)
```

and keeping it simple, no validation, we need to make our `Person` from this, we might write:

```scala
//Some function to generate an id, doesn't matter how for this example
def generateId: PersonId = ???

def makeAddress(input: AddressInput): Address = 
  Address(
    HouseNumber(input.address.houseNumber),
    StreetName(input.address.streetName),
    PostCode(input.address.postcode)
  )

def makePerson(input: PersonInput): Person = 
  Person(
    generateId, 
    input.name, 
    input.dob, 
    makeAddress(input.address)
  )
```

Chimney will allow us to make our `Address` from the `AddressInput` in a tiny amount of code, even taking care of our Value classes that we're using to avoid primitives in the model:

```scala
import io.scalaland.chimney.dsl._
def makePerson(input: PersonInput): Person = 
    Person(
      generateId, 
      input.name, 
      input.dob, 
      input.address.transformInto[Address]
    )
```

The `transformInto` function takes the output type as a type argument, and in this case, because `Address` and `AddressInput` have the same number of fields and they have the same names, and the values match up (even with Value class wrappers), a single function call does the transformation. 

Great start, but what about `Person`, it's almost the same as `PersonInput`...

```scala
def makePerson(input: PersonInput): Person = 
    input.transformInto[Person]

// Chimney can't derive transformation from chimney.ChimneyExample.PersonInput to chimney.ChimneyExample.Person

// chimney.ChimneyExample.Person
//   id: chimney.ChimneyExample.PersonId - no accessor named id in source type chimney.ChimneyExample.PersonInput


// Consult https://scalalandio.github.io/chimney for usage examples.
```
We get an error here, because `Person` has an 'id' field and Chimney doesn't know what to use for it, so we have to tell it:

```scala
def makePerson(input: PersonInput): Person = 
    input
      .into[Person]
      .withFieldConst(_.id, generateId)
      .transform
```
This api: `a.into[B].transform`, is the same as `a.transformInto[B]`, but we can provide implementations in between with a variety of functions.

`.withFieldConst()` takes a function from the Output Type to a field (in this case the Id field), and then a value to apply to that field.

So there we have it, our `PersonInput` has been changed into a `Person`, including the nested address object.


### Input validation

To take our example a little further, we might apply some validation rules to the PersonInput, such as:

- Person name can't be empty
- Date of Birth can't be in the future
- Address house number can't be less than 1
- Street name can't be empty
- Postcode must exist according to some lookup

Now we can fairly easily write this with some monadic logic, capturing errors as strings:

```scala
// Let's assume this function is able to determine whether a post code exists
def validatePostcode(candidate: String): Either[String, PostCode] = ???
  
def validateInput(input: PersonInput): Either[String, Person] = 
  for {
    name <- Option.unless(input.name.isEmpty())(input.name).toRight("Empty name")
    dob <- Option.unless(input.dob.after(new Date))(input.dob).toRight("DOB can't be in the future")
    houseNumber <- Option.unless(input.address.houseNumber < 1)(HouseNumber(input.address.houseNumber)).toRight("House number can't be less than 1")
    streetName <- Option.unless(input.address.streetName.isEmpty())(StreetName(input.address.streetName)).toRight("Empty street name")
    postcode <- validatePostcode(input.address.postcode)
  } yield Person(generateId, name, dob, Address(houseNumber, streetName, postcode))
```

Again this is reasonable, although since it uses monads (Either[String, Person] in this case), the validation is fail-fast, returning the first validation error it comes to without attempting further ones.

The problem of collecting multiple errors for this sort of computation is fairly well understood, indeed the [cats.Validated type](https://typelevel.org/cats/datatypes/validated.html) is designed for this and we can use applicative functors to collect errors. 

In fact chimney offers [cats validated support](https://scalalandio.github.io/chimney/transformers/cats-integration.html#validated-support-for-lifted-transformers)

However, for simplicity, we'll implement this just using the standard library:

```scala
// Define a type alias to capture our errors in a vector
type Validated[A] = Either[Vector[String], A]

// Assume that our post code validation function is modified to return a Validated
def validatePostcode(candidate: String): Validated[PostCode] = ???

def validate(input: PersonInput): Validated[Person] = 
  input
    .intoF[Validated, Person]
    .withFieldConst(_.id, generateId)
    .withFieldConstF(_.name, Option.unless(input.name.isEmpty())(input.name).toRight(Vector("Empty name")))
    .withFieldConstF(_.dob, Option.unless(input.dob.after(new Date))(input.dob).toRight(Vector("DOB can't be in the future")))
    .withFieldComputedF(_.address, 
      _.address
        .intoF[Validated, Address]
        .withFieldConstF(_.houseNumber, Option.unless(input.address.houseNumber < 1)(HouseNumber(input.address.houseNumber)).toRight(Vector("House number can't be less than 1")))    
        .withFieldConstF(_.streetName, Option.unless(input.address.streetName.isEmpty())(StreetName(input.address.streetName)).toRight(Vector("Empty street name")))
        .withFieldComputedF(_.postcode, a => validatePostcodeV(a.postcode))
        .transform
    )
    .transform
```

So clearly, we've lost the brevity of the transformation here, but all our validation is performed, and we collect errors into a Vector[String] meaning we'll either get a Right containing a person or a Left containing all validation errors to report back at once.


In the case of this sort of input validation use case, it's unlikely we'd use this code in more than one place, but if we did need to, we could make use of the `Transformer` type class, defining the transformation like so:

```scala
import io.scalaland.chimney._
implicit val addressTransformer: TransformerF[Validated, AddressInput, Address] = 
  Transformer.defineF[Validated, AddressInput, Address]
    .withFieldComputedF(_.houseNumber, a =>  Option.unless(a.houseNumber < 1)(HouseNumber(a.houseNumber)).toRight(Vector("House number can't be less than 1")))    
      .withFieldComputedF(_.streetName, a => Option.unless(input.address.streetName.isEmpty())(StreetName(input.address.streetName)).toRight(Vector("Empty street name")))
      .withFieldComputedF(_.postcode, a => validatePostcodeV(a.postcode))
      .transform
```

This is a little more difficult to do with the PersonInput type since we need to provide a PersonId value from externally, unless we are able to generate it within the function.

```scala
implicit val personTransformer: TransformerF[Validated, PersonInput, Person] =
  Transformer.defineF[Validated, PersonInput, Person]
    .withFieldConst(_.id, PersonId(UUID.randomUUID().toString()))
    .withFieldComputedF(_.name, p => Option.unless(p.name.isEmpty())(p.name).toRight(Vector("Empty name")))
    .withFieldComputedF(_.dob, p => Option.unless(p.dob.after(new Date))(p.dob).toRight(Vector("DOB can't be in the future")))
    .withFieldComputedF(_.address, _.address.transformIntoF[Validated, Address])
    .buildTransformer
```

Although not obvious at first, the line `_.address.transformIntoF[Validated, Address]` makes use of the `addressTransformer` so it must be in scope. 
Having the Address Transformer defined separately might come in useful if Address were to be used elsewhere in the application, such as for a Business entity or similar.

So, now that we have a person transformer defined, we can revisit our original transformation call:

```scala
def validate(input: PersonInput): Validated[Person] = 
  input.transformIntoF[Validated, Person]
```

And provided our implicit `personTransformer` is in scope, we again have a very concise simple call that can be used in our application logic without bloating it too much.

---

So we've seen how Chimney can be used to remove some of the boilerplate code needed when transforming similar models into one another.

We've also seen how we can make use of Chimney during validation to collect errors and how to use Transformers that can define complex transformations to keep our code concise in the business logic.

  