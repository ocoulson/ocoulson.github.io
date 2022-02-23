---
title: Introduction to Monads
layout: post
---

Introduction to Monads and Monadic Design in Scala

---

When starting with Functional Programming, one of the scary sounding concepts which is quite alien to many OO programmers is the dreaded 'Monad'. Even some devs who have been working with Scala for a while may not be able to fully explain the concept, even if they make use of them, which is the basis for this post really:

> _Monads may sound complex or alien, but they are quite simple to use even without understanding all the concepts behind them._

So ignoring mathsy definitions of Monads (like ~~monoids in the category of endofunctors~~), what really, for practical purposes is a Monad?

To answer this, we'll look at a few of the types provided by the Scala standard library, and anyone working in Scala has, hopefully, come across all or some of these:

- **List[T]** - _Scala's linked list implementation - a sequence of some type_
- **Option[T]** - _Scala's way of avoiding the dreaded 'null' and NullPointerException_
- **Try[T]** - _an alternative to the 'try-catch' syntax familiar to Java / JavaScript devs_
- **Either[T]** - _sort of like Try and Option, but with more freedom_

## List[T]

Let's say we have an API to query a database about authors and books:

```scala
case class Author(name: String, id: Long)
case class Book(title: String, genre: String)

def getAuthors(): List[Author] = ???

def getBooks(author: Author): List[Book] = ???
```

We want to define a function to get all books for all authors, the simplest thing to do, given our simple API, would be to get all the authors, then for each author, get all the books.

List[T] provides a `.map()` function to help us do just this, so for each element of the list of authors, we can apply the 'getBooks' function like so:

```scala
def getAllBooks(): List[List[Book]] =
  getAuthors.map(author => getBooks(author))
```

However, the return type of this function will be `List[List[Book]]` because each author becomes a list of books in the `.map()`.

So for example:

    If getAuthors returns:
    List("Charles Dickens", "Jules Verne")

    Then after the .map() we might have:
    List(
        List("Oliver Twist", "A Christmas Carol"),
        List("20,000 Leagues Under the Sea", "Around the World in 80 days")
    )

So to simplify this, we can call `.flatten`, so that we get rid of the nested list and just have a flat `List[Book]`

```scala
def getAllBooks(): List[Book] =
  getAuthors
      .map(author => getBooks(author))
      .flatten

// List(
//    "Oliver Twist",
//    "A Christmas Carol",
//    "20,000 Leagues Under the Sea",
//    "Around the World in 80 days"
// )
```

#### Enter .flatMap()

List[T] also provides a function that will do these two steps ,.map -> .flatten, in one step: `.flatMap()`

```scala
def getAllBooks(): List[Book] =
  getAuthors
    .flatmap(author => getBooks(author))
```

`flatMap()` is very similar to `map()`, but where `map()` takes as a parameter a function from any type `A => B`, flatMap must take a function from `A => List[B]`, so that all the sublists can be flattened down into a single list.

#### Chaining .flatMaps

To take this a step further, imagine we have another function to get all Reviews of a Book.

```scala
case class Review(reviewer: String, rating: Int, review: String)
def getReviews(book: Book): List[Review] = ???
```

and we want to write a function to get all reviews of all books, we need to get all authors, then for each author get all books, then for each book we get all reviews:

```scala
def getAllReviews(): List[Review] =
  getAuthors()
    .flatMap(
      author =>
        getBooks(author)
          .flatMap(
            book =>
              getReviews(book)
          )
    )
```

For each call to flatMap, we turn one element A (Author / Book) into a List of B (Book / Review), and then all those lists are flattened so that we end up with all results in a single list.

#### No results

One important thing to note, if the first of the calls to the database returns an empty list, then the flatMap function has nothing to act on, so an empty list will immediately be returned.

```scala
def getAllReviews(): List[Review] =
  getAuthors() // If this returns Nil the lines below
    .flatMap(  // do nothing and Nil is immediately returned
      author =>
        getBooks(author)
          .flatMap(
            book =>
              getReviews(book)
          )
    )
```

If one of the authors has no books, `getBooks` would return `Nil` and the flatMap on it would immediately return Nil also without making the `getReviews` call.

```scala
def getAllReviews(): List[Review] =
  getAuthors() // returns List("Charles Dickens", "Jules Verne")
    .flatMap(
      author =>
        getBooks(author) // if getBooks("Jules Verne") returned an empty List
          .flatMap(      // this flatMap would do nothing
            book =>
              getReviews(book)
          )
    )

List(List("Oliver Twist"), Nil)
// would become
List(List(List("OT - Review1", "OT - Review2")), Nil)
// and when flattened:
List("OT - Review1", "OT - Review2")
```
