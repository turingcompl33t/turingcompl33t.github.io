---
layout: post
title: "Building With C++ Coroutines: Series Introduction"
---

C++20 introduces a number of important new features: concepts, modules, ranges, a new comparison operator, etc. etc. While all of these features are interesting and deserving of study, it is the language support for coroutines that I find the most compelling.

There are a number of excellent resources on the topic of C++ coroutines that already exist and have certainly helped me arrive at my current level of understanding. Among them are:

- [Lewis Baker's Blog](https://lewissbaker.github.io/): Lewis Baker has a number of blog posts on the internal workings of coroutines, including the `awaiter` interface, the `promise` interface, and the mechanics of _asymmetric transfer_. 
- [The `cppcoro` Library](https://github.com/lewissbaker/cppcoro): `cppcoro` is a library full of beautiful coroutine abstractions, and it serves as the primary motivation for many of my experiments with coroutines. It is often after reading the interface description for a type / function in `cppcoro` that I begin wrestling with the implementation of a new coroutine-based application, primitive, or algorithm.
- [The Coroutines TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/n4775.pdf) As far as technical specifications go, the coroutines TS is remarkably readable.

My goal with this series is not to attempt to improve upon the work that the resources cited above, among others, have already completed. Rather, I hope to fill what I see as a substantial gap in the currently available resources on C++ coroutines. 

Existing resources like those available on Lewis Baker's blog do a great job of explaining the low-level details of how coroutines are implemented by the compiler and the way in which one may go about defining their own coroutines and coroutine types. 

What they don't do is go on to describe how to take these low-level coroutine primitives and use them to build useful abstractions that improve the quality and performance of our code. Of course, libraries like `cppcoro` and Facebook's `folly` already support implementations of various useful coroutine abstractions, but I found the source for these abstractions very difficult to understand when I was learning how to use coroutines myself. Thus, I'm hoping that my experiments and accompanying explanations can serve as a sort of "bridge" between the initial understanding of the low-level mechanics needed get a coroutine compiling and the high-level, generic abstractions that exist in these libraries.

The above paragraph implies that the posts in this series will assume a basic familiarity with coroutines and the requirements of coroutine types; you should know (approximately) how the `awaiter` and `promise` interfaces work and the mechanics of the methods required to implement them.