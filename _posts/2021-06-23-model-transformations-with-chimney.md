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

Say we have a REST API POST endpoint to create a person, sending a Json that looks like this, minus the ID which we'll generate in our code:

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


