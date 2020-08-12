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

First up, we'll look at the implementation of `awaitable_timer` on Linux using `epoll` and the `timerfd` API. The code for this portion of the post is located in the [`nix-epoll`](https://github.com/turingcompl33t/coroutines/tree/master/applications/timers/nix-timerfd) directory on Github.

This is not intended to be a post discussing the details or advanced usage of either of these Linux-specific interfaces (there are many great resources on this topic already), but we'll quickly cover the background necessary to understand the timer implementation.

The `epoll` API is Linux's native IO multiplexing interface. It is similar to `poll` in that it allows one to simultaneously monitor many open file descriptors for activity, but it scales much more effectively to large numbers of file descriptors because some significant differences in the internals of the implementations.

The core `epoll` API consists of just function function calls:

- `epoll_create()` or `epoll_create1()`: This function creates a new `epoll` instance and returns a file descriptor that referes to this instance. The original `epoll_create()` accepts a _hint_ that specifies the expected number of file descriptors that will be monitored by this `epoll` instance. However, this hint is ignored by all Linux kernel versions later than 2.6.8. For this reason, the `epoll_create1()` function was introduced which no longer accepts a hint and instead allows the caller to specify flags that modify the behavior of the created instance.
- `epoll_ctl()`: This function allows us to manipulate the _interest list_ for a specified `epoll` instance. Supported operations include adding file descriptors to and removing file descriptors from the interest list as well as modifying the context associated with a particular file descriptor on the interest list.
- `epoll_wait()`: This function allows us to wait for readiness events on any of the file descriptors on the interest list for a particular `epoll` instance. We pass an array of `struct epoll_event` structures and upon return these structures are populated with information for each ready file descriptor including the event that made the file descriptor ready (e.g. readable, writeable) and the context associated with the descriptor at the time the descriptor was added to the interest list.

The `timerfd` allows us to create system timers that are represented by file descriptors. The only two `timerfd` API functions that we need to know for this post are the following:

- `timerfd_create()` This function creates a new timer instance and returns a file descriptor that refers to this timer. At the the time of creation, a timer is not set to expire at any point in the future.
- `timerfd_settime()`: This function arms a timer created by a previous call to `timerfd_create()` such that it will expire at some point in the future. The API allows for specification of the expiration time in either absolute or relative terms, but we will only consider relative delays in this post. Furthermore, the `timerfd_settime()` function also allows us to specify if we would like the timer to be periodic or expire only once. Because our ultimate goal is to construct a timer type upon which we can `co_await` and such an operation is one-off delay of execution, we are not interested in periodic timers for the purposes of this post.

The `timerfd` API integrates beautifully with `epoll` because it allows us to the register each timer in which we are interested with the `epoll` instance via the standard `epoll_ctl()` call. Whenever a timer expires, the timer becomes "ready" and a thread currently monitoring for readiness events via `epoll_wait()` will be woken up. 

The program `vanilla.cpp` demonstrates basic use of the `epoll` and `timerfd` APIs to implement a basic system for executing select code after a specified delay.

The setup should be familiar after reading the preceding paragraphs. First we create both the `epoll` instance and the timer that the `epoll` instance will monitor. In the code below, the `unique_fd` type is simply a C++ RAII wrapper around UNIX-style file descriptors.

```c++
auto instance = unique_fd{::epoll_create1(0)};
if (!instance)
{
    throw system_error{};
}

auto timer = unique_fd{::timerfd_create(CLOCK_REALTIME, 0)};
if (!timer)
{
    throw system_error{};
}
```

The next thing we need to consider is the context that we want to associate with the file descriptor for our timer when we add it to the interest list for the `epoll` instance. The context that we specify will be made available to us at the time the timer expires as it is "attached" to the timer file descriptor in the `epoll` interest list and is subsequently returned to us via a successful call to `epoll_wait()`. For the purposes of this example, all we want to do when a timer expires is invoke a callback that is capable of re-arming the timer. Thus, our context consists of a function pointer to a callback function and the argument that is supplied to the callback which is simply the underlying file descriptor for the timer.

```c++
// at global scope
using expiration_callback_f = void (*)(int);

struct expiration_ctx
{
    int                   fd;
    expiration_callback_f cb;
};

// in main()
expiration_ctx ctx{ timer.get(), &on_timer_expiration };
```

Now that we have the context for the timer specified, we can register the timer with the `epoll` instance. When a `timerfd` timer expires, the file descriptor is considered "readable", so when adding the timer to the `epoll` interest list we specify that we are interested in readable-readiness on this file descriptor (`EPOLLIN`).

```c++
struct epoll_event ev;
ev.events   = EPOLLIN;
ev.data.ptr = &ctx;

if (::epoll_ctl(instance.get(), EPOLL_CTL_ADD, timer.get(), &ev) == -1)
{
    throw system_error{};
}
```

The final two things that we do in the body of `main()` are arm the timer to expire in 2 seconds and begin running the reactor. `arm_timer()` is really just a wrapper around `timerfd_settime()` that arms the timer to expire once (i.e non-periodically) after the specified delay.

```c++
template <typename Duration>
void arm_timer(int timer_fd, Duration timeout)
{
    using namespace std::chrono;

    auto const secs = duration_cast<seconds>(timeout);

    // non-periodic timer
    struct itimerspec spec = {
        .it_interval = { 0, 0 },
        .it_value = { secs.count(), 0 }
    };

    int const res = ::timerfd_settime(timer_fd, 0, &spec, nullptr);
    if (-1 == res)
    {
        throw system_error{};
    }
}
```

Finally, in the body of `reactor()`, we monitor the `epoll` instance for timer expirations via a call to `epoll_wait()`. When a timer expires, we extract the context argument from the `struct epoll_event` that is populated by `epoll_wait()` and invoke the callback stored therein. The callback simply prints a message acknowledging that the timer fired and subsequently re-arms the timer to expire again. This process continues until the specified number of expirations is reached at which point we stop monitoring for events on the `epoll` instance and simply drop out of the loop. 

```c++
for (auto count = 0ul; count < n_expirations; ++count)
{
    int const n_events = ::epoll_wait(epoller, &ev, 1, -1);
    if (-1 == n_events)
    {
        throw system_error{};
    }

    if (ev.events & EPOLLIN)
    {
        auto& ctx = *reinterpret_cast<expiration_ctx*>(ev.data.ptr);
        ctx.cb(ctx.fd);
    }
}
```

Executing `vanilla` without any arguments runs the `reactor()` loop for 5 iterations before exiting. For each iteration, we see the message acknowledging expiration of our timer:

```
$ ./vanilla
[+] timer fired
[+] timer fired
[+] timer fired
[+] timer fired
[+] timer fired
```

Alright, now that we understand the basics of setting and waiting for system timers with `epoll` and `timerfd`, how do we integrate this coroutines to create and `awaitable_timer`?

The implementation for our `awaitable_timer` type is provided in the [`awaitable_timer.hpp`](https://github.com/turingcompl33t/coroutines/blob/master/applications/timers/nix-timerfd/awaitable_timer.hpp) header. 

The constructor for `awaitable_timer` accepts the file descriptor for the `epoll` instance that will monitor for timer expirations and the delay after which this timer should fire. In the body of the construct we simply create a new timer via `timerfd_create()` and subsequently associate the file descriptor for this timer with with `epoll` instance via `epoll_ctl()`. Notice that at this point that we do not yet arm the timer because we need to defer this operation until the point at which we `co_await` upon the `awaitable_timer` instance.

```c++
template <typename Duration>
awaitable_timer(int const ioc_, Duration timeout)
    : ioc{ioc_}
    , fd{::timerfd_create(CLOCK_REALTIME, 0)}
    , async_ctx{}
{
    // ...

    struct epoll_event ev{};
    ev.events   = EPOLLIN;
    ev.data.ptr = static_cast<void*>(&async_ctx);

    int const res = ::epoll_ctl(ioc, EPOLL_CTL_ADD, fd, &ev);

    // ... 

    auto const sec_ = duration_cast<seconds>(timeout);
    auto const ns_  = duration_cast<nanoseconds>(timeout) 
        - duration_cast<nanoseconds>(sec_);

    sec = sec_;
    ns  = ns_;
}
```

When we register the timer with the `epoll` instance we specify a pointer to the `awaitable_timer`'s `async_ctx` member as the context associated with the file descriptor. Here, `async_ctx` is simply a wrapper around the `coroutine_handle` for the coroutine that will eventually be resumed upon timer expiration. We achieve this by specifying `on_timer_expire()` as the callback function that the thread running the reactor should invoke when a timer expires and providing the `async_ctx` pointer as an argument to this callback (via an opaque `void*` pointer) which we can then use to resume the waiting coroutine.

```c++
struct async_context
{   
    stdcoro::coroutine_handle<> awaiting_coro;
};

// ...

static void on_timer_expire(void* ctx)
{
    auto* async_ctx = static_cast<async_context*>(ctx);
    async_ctx->awaiting_coro.resume();
}
```

The final interesting part of the `awaitable_timer` class is the implementation of `operator co_await()` which returns the `awaiter` type and allows our `awaitable_timer` to be `co_await`ed upon. Both `await_ready()` and `await_resume()` are no-ops. `await_suspend()` stores a reference to the awaiting coroutine in the `async_ctx` member of the `awaitable_timer` instance upon which `co_await` is invoked and subsequently arms the timer owned by this instance. This has the effect of scheduling this coroutine for resumption by a thread running the reactor after the specified delay elapses and underlying `timerfd` descriptor becomes ready. 

```c++
auto operator co_await()
{
    struct awaiter
    {
        awaitable_timer& me;

        awaiter(awaitable_timer& me_) 
            : me{me_} {}

        bool await_ready()
        {
            return false;
        }

        bool await_suspend(std::coroutine_handle<> awaiting_coro)
        {
            me.async_ctx.awaiting_coro = awaiting_coro;
            return me.rearm();
        }

        void await_resume() {}
    };

    return awaiter{*this};
}
```

The [`awaitable.cpp`](https://github.com/turingcompl33t/coroutines/blob/master/applications/timers/nix-timerfd/awaitable.cpp) program demonstrates basic usage of the `awaitable_timer` type. The body of `main()` is essentially identical to its counterpart in `vanilla.cpp` apart from the divergence where instead of creating a new `timerfd` timer manually, we simply launch the `waiter()` coroutine which handles the setup of an `awaitable_timer` for us. Notice that the `waiter()` coroutine returns an `eager_task` rather than a standard `task` because we want the body of `waiter()` to execute eagerly; the first time that `waiter()` suspends is the first time that it awaits on the `awaitable_timer` instance with `co_await timer`. 

```c++
coro::eager_task<void> waiter(
    int const           ioc, 
    unsigned long const n_expirations)
{
    using namespace std::chrono_literals;

    awaitable_timer timer{ioc, 1s};

    for (auto count = 0ul; count < n_expirations; ++count)
    {
        co_await timer;

        puts("[+] timer fired");
    }
}
```

Executing `awaitable` without arguments produces output identical to that of `vanilla`. 

### OSX: `kqueue`

### Windows: The Windows Threadpool