---
title: Introduction to Monads in Scala
layout: post
---

"How I learned to stop worrying and love the flatMap"

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
case class Book(title: String, genre: String, author: Author)
case class Review(reviewer: Reviewer, rating: Int, review: String, book: Book)
case class Reviewer(name: String, id: Long)

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

List[T] also provides a function that will do these two steps ,`.map` then `.flatten`, in one step: `.flatMap()`

```scala
def getAllBooks(): List[Book] =
  getAuthors
    .flatmap(author => getBooks(author))
```

`flatMap()` is very similar to `map()`, but where `map()` takes as a parameter a function from any type `A => B`, flatMap must take a function from `A => List[B]`, so that all the sublists can be flattened down into a single list.

#### Chaining .flatMaps

To take this a step further, imagine we have another function to get all Reviews of a Book.

```scala
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

One important thing to note is that if `.flatMap()` is called on an empty list, the function passed is not evaluated and instead an empty list is immediately returned.

So, if the first of the calls to the database returns an empty list, then the flatMap function has nothing to act on, so an empty list will immediately be returned.

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
          .flatMap(      // this flatMap would do nothing and immediately return an empty list
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

Ok, but what does this have to do with Monads and Monadic design?

Well, we've just met our first Monad. List[T] is a Monad!

Let's look at our next Monad: Option[T]

## Option[T]

It is not too much of a jump from List[T] to Option[T] - conceptually you can consider an Option[T] to be a list with a maxiumum of 1 element. Therefore it is no surprise that Option[T] has a nearly identical API to List[T].

The usage is where Option[T] differs from List[T] - An Option is used primarily to remove the need for the dreaded `null`, and provide simpler, functional ways of dealing with an operation that might return nothing or otherwise (non-exceptionally) fail.

Expanding on our example from before, let's imagine that our API for querying Authors and Books requires an API key, which we expect to be stored in a session in our application.

```scala
// This function will return an empty Option (None) if the
// Author doesn't exist or if the apiKey isn't valid
def getAuthor(name: String, apiKey: String): Option[Author] = ???

// This function will return the Api Key from the session or
// None if there is no available Api Key
def getApiKey(): Option[String] = ???
```

So in order to query for an Author, we first must get the API key, then pass it along to the API along with our name query:

```scala
def findAuthor(name: String): Option[Author] = {
  getApiKey().flatMap(
    key =>
      getAuthor(name, key)
  )
}
```

Once again `.flatMap()` allows us to first query the apikey, then pass it to the getAuthor function, returning at the end a single flat Option. `None` will be returned if:

- the Api Key is not in the session
- the Api Key is not valid
- the Api Key is valid but no author by that name exists

Let's take it further and add some complexity. We need to define a function, that given 2 Reviewers' names, will return any Books which both reviewers have recommended (rated at 7 or above), and we have the following API calls to use:

```scala
// API functions
def getReviewer(name: String, apiKey: String): Option[Reviewer] = ???
def getReviewsAboveNRating(reviewer: Reviewer, n: Int, apiKey: String): Option[List[Review]] = ???


// Let's define a helper function that given 2 lists of reviews,
// returns any books that are in both lists
def findCommonBooks(reviews1: List[Review], reviews2: List[Review]): List[Book] = ???

// Our function:
def findBooksReccommenedByReviewers(reviewer1: String, reviewer2: String): Option[List[Book]] =
  // First get the API key from the session
  getApiKey()
    .flatMap(
      apiKey =>
        // then find the first reviewer by name, which might not exist
        getReviewer(reviewer1, apiKey)
          .flatMap(
            (r1: Reviewer) =>
              // then find the second reviewer by name, which might not exist
              getReviewer(reviewer2, apiKey)
                .flatMap(
                  (r2: Reviewer) =>
                    //now we have both reviewers and they both exist, look up their reviews
                    getReviewsAboveNRating(r1, 7, apiKey)
                      .flatMap(
                        (r1Reviews: List[Review]) =>
                          //now get the reviews for the second reviewer
                          getReviewsAboveNRating(r2, 7, apiKey)
                            .flatMap(
                              (r2Reviews: List[Review]) =>
                                findCommonBooks(r1Reviews, r2Reviews)
                            )

                      )
                )
          )
    )
```

So obviously this is quite a complex set of calls, and each one could fail if the API key is invalid (although obviously the failure would probably only occur on the first call to `.getReviewer()` if it were invalid, however the API is defined this way to require the key on all calls).

If any one of these calls were to return None (because the API key is invalid for example) - the remaining chained calls would not happen, and None would be immediately returned.

You can chain all these calls because each one (`getReviewer` and `getReviewsAboveNRating`) returns an Option[T] - it returns the same Monad type each time. Each flatMap call MUST return the same Monad, so that a nested Option[Option[Option[T]]] can be flattened to a simple Option[T]

The use of chained flatMaps like this provides us with a fail-fast composition of functions, which is often what is required (although not always) and so is very useful.

This is one of the principal features of Monadic design:

> Chaining flatMap calls gives us fail-fast logic when composing multiple functions together

### For Comprehensions in Scala

Chaining flatMap alls is so widely used that in Scala we have a useful feature to make it more readable: The `for comprehension` - which is simply syntactic sugar disguising chained.

Let's rewrite our huge nested function above using a for comprehension:

```scala
def findBooksReccommenedByReviewers(reviewer1: String, reviewer2: String): Option[List[Book]] =
  for {
    apiKey <- getApiKey()
    r1 <- getReviewer(reviewer1, apiKey)
    r2 <- getReviewer(reviewer2, apiKey)
    reviews1 <- getReviewsAboveNRating(r1, 7, apiKey)
    reviews2 <- getReviewsAboveNRating(r2, 7, apiKey)
  } yield findCommonBooks(reviews1, reviews2)
```

This has exactly the same logic as the above function, but much more concise and easier to read.

Remember:

- all function calls preceded by the `<-` symbol must return the same monad, in this case: `Option[T]`
- if any of the calls (top down) return None, the remaining functions are not even attempted and None is immediately returned

Considering the fail-fast feature of the for comprehension, we could actually improve this a bit to avoid unnecessary calls.

If the first reviewer has 0 reviews above rating 7 - there is no point in even querying the second reviewer at all.

```scala
def findBooksReccommenedByReviewers(reviewer1: String, reviewer2: String): Option[List[Book]] =
  for {
    apiKey <- getApiKey()
    r1 <- getReviewer(reviewer1, apiKey)
    reviews1 <- getReviewsAboveNRating(r1, 7, apiKey) //immediately get reviews for reviewer1 if found.
    nonEmptyReviews <- Option.unless(reviews1.isEmpty)(reviews1) // ensure reviews1 is non-empty - or return None

    // if reviews1 is empty, the below lines don't even get evaluated.
    r2 <- getReviewer(reviewer2, apiKey)
    reviews2 <- getReviewsAboveNRating(r2, 7, apiKey)
  } yield findCommonBooks(reviews1, reviews2)
```

Let's also look back at our nested flatMaps with List[T]s we saw above, and rewrite that with a for-comprehension:

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

def getAllReviews2(): List[Review] =
  for {
    author <- getAuthors()
    book <- getBooks(author)
    review <- getReviews(book)
  } yield review
```

With the lists, the functionality is not really 'fail-fast' but instead 'ignore emptys' as a call returning 0 results isn't really a failure. But ultimately the purpose is the same,

### What's a flatMap

So what is a flatMap function?

A function defined in a container type such as List / Option, which takes a function from the contained type A to another container containing another type B (which could be the same as A)

```scala
trait List[A] {
    def flatMap[B](f: A => List[B]): List[B]
}

trait Option[A] {
    def flatMap[B](f: A => Option[B]): Option[B]
}
```

### What's a monad?

And finally, what is a Monad?

A Monad is a data structure that contains another type (with a type parameter), and that defines a `.flatMap()` function allowing you to compose calls in a sequential / fail-fast manner.

```scala
trait Monad[A] {
    def flatMap[B](f: A => Monad[B]): Monad[B]
}
```

continued in part 2
