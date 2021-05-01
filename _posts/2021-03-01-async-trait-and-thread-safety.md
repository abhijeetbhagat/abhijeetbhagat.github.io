---
layout: post
title:  "Async traits and thread-safety"
---

As of writing this article, Rust has no out-of-the-box support for traits with async functions. For this, the `async-trait` crate has to be used:
&nbsp;

```Rust
use async_trait::async_trait;

#[async_trait]
trait Foo {
    async fn baz(&self);
    async fn bar(&mut self);
}
```
&nbsp;
Let's say we are providing a default implementation for the `bar()` method above like:
&nbsp;
```Rust
...
    async fn bar(&mut self) {
        self.baz().await;
    }
...
```
&nbsp;
This doesn't work:
&nbsp;
```Rust
error[E0277]: `Self` cannot be shared between threads safely
...
   |
   |       async fn bar(&mut self) {
   |  _____________________________^
   | |         self.baz().await;
   | |     }
   | |_____^ `Self` cannot be shared between threads safely
   |
   = note: required because of the requirements on the impl of `Send` for `&Self`
   = note: required because it appears within the type `for<'r, 's, 't0> {ResumeTy, &'r mut Self, &'s Self, Pin<Box<(dyn Future<Output = ()> + Send + 't0)>>, ()}`
   = note: required because it appears within the type `[static generator@src/main.rs:8:29: 10:6 for<'r, 's, 't0> {ResumeTy, &'r mut Self, &'s Self, Pin<Box<(dyn Future<Output = ()> + Send + 't0)>>, ()}]`
   = note: required because it appears within the type `from_generator::GenFuture<[static generator@src/main.rs:8:29: 10:6 for<'r, 's, 't0> {ResumeTy, &'r mut Self, &'s Self, Pin<Box<(dyn Future<Output = ()> + Send + 't0)>>, ()}]>`
   = note: required because it appears within the type `impl Future`
   = note: required because it appears within the type `impl Future`
   = note: required for the cast to the object type `dyn Future<Output = ()> + Send`
```
&nbsp;
From the `async-trait`'s docs:
> The one wrinkle is in traits that provide default implementations of async methods. In order for the default implementation to produce a future that is Send, the async_trait macro must emit a bound of Self: Sync on trait methods that take &self and a bound Self: Send on trait methods that take &mut self.

&nbsp;
Since we have a default implementation which accepts an `&mut self`, our trait should be deriving from both `Send` and `Sync`:
&nbsp;
```Rust
#[async_trait]
trait Foo: Send + Sync {
    ...
}
```
&nbsp;
And this solves the problem.
