---
layout: post
title: Coroutine Composition
---

This blog post explores the semantics of coroutine composition by examining a simplified implementation of the `task` type.

### Disclaimer

As suggested in the above summary, the implementation of `task` explored in this post lacks a number of key features that would be necessary for it to see use in non-trivial code. First and foremost, our `task` does not support returning values upon task completion. I made the decision to omit return values from this `task` implementation because I find that the necessity of dealing with generics (via templates) in the implementation distracts from the primary objective of the post which is to understand coroutine composition. 

Another feature missing from this implementation of `task` is thread safety - it is _not safe_ to utilize this `task` type in a multithreaded program. We will look at the potential races inherent in this limited implementation later on in the post.

For a more complete, production-quality `task` implementation, see the [`task.hpp`](https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/task.hpp) header in the [cppcoro library](https://github.com/lewissbaker/cppcoro) library on Github.

### Introduction: What is a `task`?

### Basic Usage of the `task` Type

### Composing Coroutines with `task`: A First Try

### Composing Coroutines with `task`: A Corrected Implementation

```c++
void await_suspend(std::coroutine_handle<> awaiting_coro)
{
    coro_handle.promise().continuation = awaiting_coro;
    coro_handle.resume()
}
```
