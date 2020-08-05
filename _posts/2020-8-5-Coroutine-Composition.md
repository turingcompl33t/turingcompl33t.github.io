---
layout: post
title: Coroutine Composition
---

This blog post explores the semantics of coroutine composition by examining a (simplified) implementation of the `task` type.

```c++
void await_suspend(std::coroutine_handle<> awaiting_coro)
{
    coro_handle.promise().continuation = awaiting_coro;
    coro_handle.resume()
}
```
