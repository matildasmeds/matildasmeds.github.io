Fun with functions: Exploring Rust function pointers and traits

Being skilled in something, involves feeling at ease with the tools. A practical nimbleness, certain effortlessness. That to me is craft. This sort of intuitive understanding of the language may be elusive, but one can definitely take steps towards it. And the closer one gets, the more easily we can focus on the actual task at hand, instead of just struggling with the tools. Over time we may move to solve more challenging problems, or at least solve the same old problems better. Tiny experiments can help build that skill. 

Like the marketers of BlendTec, asking "Will It Blend?", I find myself asking, "Will It Compile?" Sometimes it does, sometimes it doesn't, and in either case one learns something. 

The Rust syntax is rich and interesting and function pointers are no exception. In this post we will explore different ways of returning and passing functions, and a little bit of generics and trait bounds as well. This post is intended for someone who may be new to Rust, and has some previous programming experience.

fn() - function pointer

The simplest option is to return a function pointer with fn(). It is stateless, i.e. it does not capture values from the environment. They can be resolved at compile time, use static dispatch, and don't require heap allocation.

Let's see what it could look like. Let's imagine a nice little enum called Operation.

use std::ops::{Add, Mul};

enum Operation {
    Add,
    Multiply,
}

impl Operation {
    fn apply<T: Add<Output = T> + Mul<Output = T>>(&self, a: T, b: T) -> T {
        match self {
            Operation::Add => a + b,
            Operation::Multiply => a * b,
        }
    }
}

Here we implement a generic apply() function for the enum, with the trait bound that any type T is accepted, as long as T + T is defined and results in T, and the same for multiplication. This is actually pretty cool, we get a generic function with a limited amount of code.

But, maybe I want to return the operation as a function instead, while keeping it as generic.

use std::ops::{Add, Mul};

enum Operation {
    Add,
    Multiply,
}

impl Operation {
  fn as_function<T: Add<Output = T> + Mul<Output = T>>(&self) -> fn(T, T) -> T {
      match self {
          Operation::Add => |a: T, b: T| -> T { a + b },
          Operation::Multiply => |a: T, b: T| -> T { a * b },
      }
  }
}

Here we have a function that returns a function. A bit more code, and I find the previous version more readable. Yet, it's a useful exercise, to gain more nimbleness with Rust.

What if we want to take a function in as a parameter, can I do it with fn()? Absolutely.

fn apply<T>(fun: fn(T) -> T, param: T) -> T {
    fun(param)
}

fn main() {
    // This will work
    println!("{}", apply(|x| { 3 * x}, 4));
    // This will not work, because the closure captures environment. Try it!
    // let a = 1;
    // println!("{}", apply(|x| { a * x}, 4));
}

As seen, fn() allows us to take in a function as a parameter and return it using the closure syntax.
If you want to, you can open Rust Playground https://play.rust-lang.org/, build these examples or write your own, and add a main method or tests to verify behavior. Here's also Rust Book on closures: https://doc.rust-lang.org/book/ch13-01-closures.html


fn() is the easiest way to pass closures around. However, there is more!

What should we do, if we want to pass a closure that captures environment? Or offer the developer using our code the opportunity to choose?

Function traits: FnOnce, Fn, FnMut

Especially for library code, using Fn-traits (FnOnce, FnMut, Fn, with FnOnce including the two others as well) can be considered more user friendly, than using fn().

Let's translate our previous example to use FnOnce-trait.

fn apply<T>(fun: impl FnOnce(T) -> T, param: T) -> T {
    fun(param)
}

// Alternative syntax for the same thing
fn apply2<T, F>(fun: F, param: T) -> T
   where 
    F: FnOnce(T) -> T {
  fun(param)
}

// Not bad, still pretty readable
// Now we can do this....

fn main() {
    let a = 1;
    println!("{}", apply(|x| { a * x}, 4));
    println!("{}", apply(|x| { 3 * x}, 4));
}

So to recap, we have...
fn() static dispatch, no capturing of the environment
Impl Fn() static dispatch, allows capturing environment.
The latter can be considered more user friendly, as it allows for passing both types of closures.

I've always felt that the function pointer is easier, but looking at these examples, there is not that much of a difference!

Now that's pretty neat. But this is Rust, so there is more!

Next part: Box<dyn Fn> What is this and why use it?


https://doc.rust-lang.org/book/ch19-04-advanced-types.html
