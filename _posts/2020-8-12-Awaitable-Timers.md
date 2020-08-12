---
layout: post
title: "Awaitable System Timers"
---

Coroutines enable various interesting asynchronous programming constructs on their own; `co_await`able synchronization primitives are one such example. However, the power of coroutines really comes through when we integrate them with some external event source. In this post we will look at doing just by constructing an `awaitable_timer` type that utilizes the underlying system's native timer facilities to delay resumption of the coroutine until after some delay has elapsed.

Naturally, the timer facilities available to our program vary widely based on the operating system on which our program runs. In order to get a feel for how coroutines may be integrated with different OS APIs, I've implemented the `awaitable_timer` construct on three different platforms utilizing each system's native API for asynchronous event management:

- Linux backed by `epoll` and `timerfd`
- OSX backed `kqueue`
- Windows backed by the Windows Threadpool

### Linux: `epoll` and `timerfd`

### OSX: `kqueue`

### Windows: The Windows Threadpool