+++
title = 'Learning Rust Lifetimes'
date = 2024-01-07
+++

There comes a time in each Rustacean's life, when one wants to get a better grasp on what lifetimes are and how they work. This post is written for *that* moment. I will share what I have learnt about lifetimes, and how to apply some deliberate practice on the syntax.

> If you notice any room for improvement or have further questions about lifetimes, let me know. My sincere aim is to share something correct, accessible and hopefully useful, and keep learning.

Lifetime (of a reference) is one of those central Rust concepts, that stem directly from its memory model. I have not seen anything quite like it in other programming languages.Having a good mental model of your tools is crucial. There are productivity and confidence gains that can only be achieved by improving your mental models.  

For lifetimes, a foundational knowledge would include understanding why and when they are needed, language syntax, and being able to apply lifetime annotations, in the typical cases where they are needed.

Lifetimes definitely add to the learning curve of Rust, and can feel even a bit scary sometimes.  By investing a little bit of time into learning them, you will find that working with Rust becomes easier and more effective. 

## Defining lifetimes

The gist of it: Rust compiler uses lifetimes to ensure that references are valid.

In Rust, every value has a scope, for the span of which we can access the value. Owned values are managed by Rust's ownership system, and they are dropped after they go out of scope. In Rust all references have a lifetime. They describe how the lifetime of the reference relates to other lifetimes and scopes. 

Lifetimes can be thought of as contracts for your references, that the compiler uses to ensure memory safety. With memory safety we mean managing the physical memory adequately: Freeing memory that is no longer used, making sure the memory is not accessed after freeing it, and also making sure that the memory segments' contents match to the types defined by the programmer.

Lifetimes feel a bit elusive, and I think this is for two reasons. First, they seem unfamiliar. I haven't seen anything quite similar in other programming languages I have used. Second, they are relative, rather than exact. Thankfully so, if the programmer had to define manually up until where in the code a reference should be kept, we would be back in the [segfault](https://en.wikipedia.org/wiki/Segmentation_fault) corner, trying to manage memory manually. Instead, lifetimes are relative. They communicate the intent of the programmer, and let the compiler do the right thing.
## Lifetime elision

Most of the time you don't have to worry about lifetimes at all, as the compiler resolves them on its own, using what we call lifetime elision. Lifetime elision is the set of rules the Rust compiler uses to deduce how long references should live.  Lifetime elision is so powerful, that it is possible to write quite a lot of Rust without really knowing much about lifetimes at all.

When lifetime elision rules apply, you won't need to define lifetimes manually. When they do not apply (or don't alone lead to semantical correctness), that is when you need to wrap up your sleeves and add them in your code.

With lifetime elision rules and lifetime annotations (when needed) the compiler ensures that the reference does not outlive the data it points to.

### Lifetime elision rules are powerful, even if they are just a few

#### Functions
1. Each parameter that is a reference, is assigned its own lifetime
2. If there's exactly one input parameter lifetime, the same lifetime is assigned for all output references.
3. If there are multiple input parameters, but one of them is `&self`, or `&mut self`, that lifetime is assigned to all output references.
#### Structs
When a struct has reference fields, you will need to specify their lifetimes explicitly.

### When lifetime elision leads to semantical incorrectness
**TODO: Link to the post about when lifetime elision leads to semantical incorrectness, after all, that is fascinating!**

**TODO: Should I publish this in two parts?**

## Syntax for annotating lifetimes

TODO!
I can also write this based on what I discover in the exercises...

## Practice makes perfect

We have covered lifetimes on a theoretical level. Now it is time for practice. Here are a few exercises that can help you to learn how to use lifetimes. For best outcome, try to write actual code for each exercise, and make sure the code compiles and works correctly.

The craft of a programmer, even in the age of generative tools, is knowing the syntax very well, preferably by heart: being able to read it fluently, type it when needed, and with generative tools, check the correctness of the suggestions.

Let's start!
### Basic lifetime annotation for functions

Write a function that takes two string slices and returns a reference to the longest one.
1. First variation: Use the same lifetime for both input parameters
2. Second variation: Use a different lifetime for different input parameters

This exercise is inspired by the examples in The Rust Programming Language Book: [Validating References with Lifetimes](https://doc.rust-lang.org/stable/book/ch10-03-lifetime-syntax.html)

We could use references to any other type as well. String slices are easy to create though, and Rustaceans tend to be familiar with them.

**TODO: A base function for both
TODO: Solutions (Available at the bottom of the page)**
