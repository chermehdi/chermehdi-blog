+++
title = "Why I like rust"
date = 2020-07-29
author = "Mehdi Cheracher"
tags = ["rust"]
keywords = ["rust", "rust programming"]
description = "A tour of the rust language features, and why you should consider it"
showFullContent = false
+++

## Introduction

- About a year a go I stumbled on a conference presenting the Rust programming
  language, and was intrigued by it's offerings, So I decided that probably
  I should pick it up and see if I can apply it to some use cases I had in mind,
  I only started doing it months later, and after picking the Rust book and
  reading it, doing some example projects, and fiddling with it using some projects I wanted to try, 
  I decided that it's time to give my feedback on it in this post.


**Disclaimer**: This post is not a Rust tutorial, it's more of a *Motivation to use Rust* and
  What's my opinion / experience using the language in the last couple
  of months, if you want some good resources on learning it, check the resources
  section I'll make sure to include some useful links.

### Some distinction of the rust programming language.

  Rust is a Systems programming language that was created by Mozilla, it promises things
    such as **Fearless concurrency**, and **Compile-time thread safety**, one of the nice things about it is that although it was originaly create at Mozilla
    , it's development is completely open source driven.
 Rust is a compiled language, with a **very** strong and elegant type system, it's essentially an imperative language, but it comes with some nice functional aspects like
  the ones you can find in Haskell or Scala. 
 Rust does not need any runtime, and it does not have a garbage
    collector (more on this later).

### Compared to Go
I will try to do a quick comparison between Rust and Go since these two are
  the two recent languages I learned, and they are quite popular these days. 
  - Go has a garbage collector which implies GC pauses.
  - Rust does not have `nil` (or equivalent) pointers.
  - Rust provides nicer error handling via its rich type system.
  - Safe concurrency at compile time, although Go provides easier concurrency
  primitives, it is easy to shoot yourself in the
  foot if you use them incorrectly (dead locks, concurrent modifications errors
  ...).
  - Rust provides `Zero cost abstractions` and has a Stronger type system.
  - One deal breaker for me when using Go was ** Generics ** and the support
  for them in Rust is excellent.
  - Dependency management feels easier in Rust, With cargo you can do most of
  the things you can do with the Go build system, but you can also do more.
  ...

### Rust major selling points

#### Safety

- It is very **hard** to hurt yourself using Rust, the compiler will practically
fight you very hard on a lot of things that you think should work
especially if you are new to the language, But if you win, your programme
is guaranteed to be Safe.

Rust provides safety guarentees such as Pointer checking, thread safety ... and
all at compile time, So how is this done?

```rust
let a = Vec::new();
let b = a;
drop(a); // illegal b is the current owner
```

In the code above, the call to `drop` is similar to calling the destructor
of a class in `C++`.

Rust comes with a new way of thinking about Programming in general, **The Ownership model**.
New people trying Rust for the first time struggle a lot with understanding
why certain operations are not permitted, but the gist is one should
understand the two major concepts **Ownership** and **Borrowing** and most of
the error messages thrown by the compiler will start making sense, but in
general Rust guarantees using this model that there are no dangling
pointers, nor use-after-free errors at **Compile time** and will complain at
every possible trigger of any of these errors.

Rust took the correct choice regarding mutability, everything is **immutable**
by default, if you want to mutate some value, you need to make sure that
you explicitly mark it as mutable, and rust will ensure that there can
only be a single mutable reference to a value, even across thread
boundaries, which simply eliminate possible errors coming from data races when multiple
threads trying to update the same value.

#### Generics, Algebraic data types and pattern matching

- Generics in Rust are similar to the ones in `C/C++` and look like Generics in `Java` but they behave differently. Let's have a look at some code:

```rust
fn find_all<T, P>(v: Vec<T>, predicate: P) -> Vec<T>
where
    P: Fn(&T) -> bool,
{
    let mut result = Vec::new();
    for item in v {
        if predicate(&item) {
            result.push(item);
        }
    }
    result
}
```

This looks familiar coming from `Java` or `C++` but the nice thing about this
is how Rust treats it, whenever we call this function with a combination
of concrete types, Rust compiler will make sure that the `where` constraints
on the types are met (in this example we require that the `P` type is a function that takes a 
reference to a `T` and returns a `bool`), but also it will compile it as a new function as if the
generics were not there, (if you have a lot of generic functions with
different callsites this can lead to a bigger binary, but it removes the
Runtime overhead of Polymorphic callsites found in languages such as Java). 

Algebraic data types are types that are made by composing other data types, these are more
familiar to people coming from Haskell or some other functional language, in
Rust they are heavily used to provide nicer abstractions over common
behaviour, let's look at an example.

By far one of the two most used data type in Rust are in fact algebraic datatypes,
the `Option` and `Result` types, The `Option` type is used when an
operation can compute some data and returns it, or it can't, and therefore
returns nothing, it's defined in the standard library roughly like this:  

```rust
pub enum Option {
  Some(T),
  None,
}
```

The nice things about this + There are no `NULL` pointers, when a function
is written like: 

```rust
pub fn do_something(a: u32) -> Option  {
...
}
```

The caller should handle the return value as an option, thus the caller
cannot forget to check if the value can be computed or not, and therefore,
you get safer code at compile time.
This also gives birth to some nice syntactic features, such as: 

```rust
if let Some(res) = do_something(42) {
  // This will get executed if a value is returned. 
  // i.e this will not execute if the do_something function returned None.
}
```
or 
```rust
let res = match do_something(42) {
  Some(u) => u,
  None => default_value,
}
// res here will always contain some value that you can work with.
```
- And What's even cooler is that the match statement is not just a syntactic
sugar, Imagine writing a new Algebraic data type with 3 possible outcomes
and used your match statement as shown in the example above, after a while
another developer added another possible outcome to that data type, and
forgot to handle it in all of the places using this type, in another
language this will only surprise you at runtime, at some code path, but with
Rust, your code won't even compile, and the compiler will complain that you
didn't handle all the outcomes and that you should handle it, i.e if your
`match` is not exhaustive your code does not compile, How cool is that.
There is more to this section, You can read more in the Book linked below.

#### Modern tooling 

I think the toolchain that comes with Rust is quite impressive, and I'll try to explain why: 
- It's unified: All rust `crates` uses the same build system `Cargo`. A lot
    of the pain of having to deal with a language with heterogenous
    ecosystem with a lot of build systems just goes away.
- `Cargo` knows about Tests and Docs: Testing is built in into the
  language, and most of the crates that you will find on the internet has
  some form of unit tests, because it's easy to write and integrate well
  with your existing code.
  
  ```rust
  /// This is a doc test
  ///
  /// ```
  /// assert_eq!(1, get_one(1));
  /// ```
  fn get_one() -> usize {
    1
  }
  #[test]
  fn test_get_one() {
    assert_eq!(1, get_one());
  }
  ```
  Documentation is also recognized, and `Cargo` can generate the
  documentation of all your crate and link to types in the standard
  library with their documentation, or types from third party dependencies
  in a seemless manner.
  
  The language also supports a very cool feature, `doc tests`, your
  examples in the documentation of your `crate` are treated as tests, and
  if after some update they don't work (breaking API change), the build
  fails automatically, which forces the documentation to be always up to
  date.

#### Low level code
- Rust is a System's programming language, This is somewhat vague, and does not
  mean that you can't use it to build your next web app, but it mainly means
  that's it's suited to be deployed on machines with no dependencies, Rust does
  not need a Runtime, GC nor an OS for that matter, you can run Rust on embedded
  devices, create your own operating system, write device drivers ... with all
  the safety guarantees that the language provides (well most of the time...).

## Conclusion:
- Rust is a very interesting language, with a lot of potential, the community is
  still small (yet very active), the adoption is getting better, and I think we
  will see a lot more of Rust in industry in the years to come.
- On the other hand Rust can be quite a pain to learn even if you are an
  experienced developer is some other language, and it's even harder for people
  with no programming experience, that's why I think other similar languages such as Go got adopted quickly, 

## Resources:

In this section I tried to compile a bunch of links that might help when trying
to learn/understand the things mentioned above, feel free to read at your own
pace.
- [The Rust book](https://doc.rust-lang.org/book/)
- [The Rust cookbook](https://rust-lang-nursery.github.io/rust-cookbook/)
- [Zero cost abstractions](https://blog.rust-lang.org/2015/05/11/traits.html)
- [Crates repository](https://crates.io)
- [Collection of exercises](https://github.com/rust-lang/rustlings)
- [Collection of resources for more idiomatic rust
  code](https://github.com/mre/idiomatic-rust)

