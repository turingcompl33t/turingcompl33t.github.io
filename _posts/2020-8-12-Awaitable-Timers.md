---
layout: post
title: "Awaitable System Timers"
---

Coroutines enable various interesting asynchronous programming constructs on their own; `co_await`able synchronization primitives are one such example. However, the power of coroutines really comes through when we integrate them with some external event source. In this post we will look at doing just by constructing an `awaitable_timer` type that utilizes the underlying system's native timer facilities to delay resumption of the coroutine until after some delay has elapsed.

### The Interface

```c++
task<void> async_work()
{
    awaitable_timer two_seconds{2s};

    // do some work

    // suspend this coroutine for two seconds
    co_await two_seconds;

    // do more work
}
```

### Different Systems, Different Timers

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

Next up is an implementation of `awaitable_timer` on OSX backed by `kqueue`. 

`kqueue` is the native event-notification system on several operating systems including OSX and many BSD variants. Like `epoll` on Linux, `kqueue` is a readiness-based system in which we (application programmers) register interest in kernel events and then are able to wait on notification from the kernel when these events occurs. However, `kqueue` presents a slightly more general interface by introducing the notion of a `kevent`s - a tuple of a unique identifier, a _filter_ that describes the events of interest for the identifier, and (optionally) some user-provided context to associate with the identifier. Internally, the `kqueue` system uses the filter for each `kevent` to determine if the `kevent` is "ready" and, if it is, inserts the `kevent` onto the `kqueue` so that application code can later retrieve it.

The core API for `kqueue` consists of just two system calls:

- `kqueue()`: This call constructs a new `kqueue` instance and returns a file descriptor that refers to this instance. 
- `kevent()` This call is used to both register new `kevent`s with the `kqueue` instance as well as query any notified `kevent`s that are already associated with the instance. In a way, the `kevent` system call behaves like a combination of both `epoll_ctl()` and `epoll_wait()`.

Another major difference between the `epoll` API and the `kqueue` API is that `kqueue` has 'built-in" support for timers in the form of a timer filter. That is, whereas with `epoll` we had to use the `timerfd` API to construct a timer, associate this timer with the `epoll` instance, and subsequently wait for readiness of the timer via a call to `epoll_wait()`, with `kqueue` we need only specify the timer filter (`EVFILT_TIMER`) in a call to `kevent()` to add a new timer `kevent` to the kernel-maintained `kqueue` instance. Note that if you search for the OSX `kqueue` API (e.g. via Google) the Apple developer manual pages result erroneously states that the `EVFILT_TIMER` filter is not supported on OSX. 

As in the `epoll` example, the program `vanilla.cpp` demonstrates basic usage of the `kqueue` API to wait for system timer events. The setup is very similar to that used in the `epoll` example, so we'll only cover the `kqueue`-specific divergences here.

First, we create a new `kqueue` instance via the `kqueue()` system call.

```c++
auto instance = unique_fd{::kqueue()};
if (!instance)
{
    throw system_error{};
}
```

Next, we use the `kevent()` system call to register a new `kevent` with the `kqueue` instance. Here, `ident` is simply a unique identifier for the event that is completely at our discretion. The `EVFILT_TIMER` member of the structure is the built-in filter that was described above, the `EV_ADD` flag specifies that we are registering new interest in this `kevent` (note that `EV_ADD` also implicitly adds the `EV_ENABLE` flag), and the `NOTE_USECONDS` setting for the `fflags` member specifies that the timeout that we specify in the `data` member is in microseconds. We do not utilize the optional `udata` member of the structure to pass additional context to the `kevent`. 

```c++
struct kevent ev = {
    .ident  = ident,
    .filter = EVFILT_TIMER,
    .flags  = EV_ADD,
    .fflags = NOTE_USECONDS,
    .data   = us.count(),
    .udata  = nullptr
};

int const result = ::kevent(instance, &ev, 1, nullptr, 0, nullptr);
if (-1 == result)
{
    throw system_error{};
}
```

The default behavior with the `EVFILT_TIMER` filter is to establish a periodic timer. This implies that the code above registers a `kevent` that will be inserted into the `kqueue` every `us.count()` microseconds until we disable manually.

Finally, we use the `kevent()` system call once more to wait for notification that the timer `kevent` has been inserted into the `kqueue`.

```c++
struct kevent ev{};

for (auto i = 0ul; i < n_reps; ++i)
{
    int const n_events = ::kevent(instance, nullptr, 0, &ev, 1, nullptr);
    if (-1 == n_events)
    {
        puts("[-] kevent() error");
        break;
    }

    if (ident == ev.ident)
    {
        puts("[+] timer fired");
    }

    // something strange happened...
}
```

The overall structure of this program should feel very similar to that presented in the previous section with `epoll`. Running the `vanilla` executable without arguments goes through the steps above, waits for five notifications of the timer, and finally breaks the reactor loop and cleans up.

```c++
$ ./vanilla
[+] timer fired
[+] timer fired
[+] timer fired
[+] timer fired
[+] timer fired
```

Constructing an `awaitable_timer` type on top of this API is straightforward; the implementation closely resembles that presented in the `epoll` example. Specifically, the `operator co_await()` member of the `awaitable_timer` is nearly identical to the one we saw previously:

```c++
// awaitable_timer
auto operator co_await()
{
    struct awaiter
    {
        awaitable_timer& me;

        awaiter(awaitable_timer& me_)
            : me{me_} {}

        bool await_ready() { return false; }

        bool await_suspend(stdcoro::coroutine_handle<> awaiting_coro)
        {
            me.async_ctx.awaiting_coro = awaiting_coro;
            return me.arm_timer();
        }

        void await_resume() {}
    };

    return awaiter{*this};
}
```

The only distinction here is in the implementation of `arm_timer()`. Whereas in the `vanilla.cpp` program we relied on the periodic nature of the timer `kevent`, here we only want the timer to fire once and not be re-inserted into the `kqueue` until we arm it again when the timer is `co_await`ed upon. We accomplish this behavior by specifying the `EV_ONESHOT` flag when we register the `kevent` with the `kqueue` instance:

```c++
// awaitable_timer
bool arm_timer()
{
    struct kevent ev = {
        .ident  = ident,
        .filter = EVFILT_TIMER,
        .flags  = EV_ADD | EV_ONESHOT,
        .fflags = NOTE_USECONDS,
        .data   = timeout.count(),
        .udata  = &async_ctx
    };

    int const result = ::kevent(ioc, &ev, 1, nullptr, 0, nullptr);
    return (result != -1);
}
```

Usage of the `awaitable_timer` type is identical to that shown in the `epoll` example.

```c++
awaitable_timer timer{ioc, 0, 3s};

for (auto i = 0ul; i < n_reps; ++i)
{
    co_await timer;

    puts("[+] timer fired");
}
```

### Windows: The Windows Threadpool

There are a number of viable ways to create waitable system timers on Windows, and these different mechanisms significantly influence the approach that we take in making these timers compatible with coroutines.

The most obvious method for creating timers on Windows is via the [waitable timer interface](https://docs.microsoft.com/en-us/windows/win32/sync/waitable-timer-objects). However, some issues arise when attempting to integrate Windows waitable timers with an event multiplexing interface such as IO completion ports. It is possible to address these issues and utilize this approach successfully to build awaitable system timers, but it requires a significantly more involved implemention. We'll take a look at this approach in the subsequent section. For now, we'll look at a simpler approach that utilizes the Windows threadpool.



In light of this difficulty, we'll explore an alternative method here: Windows threadpool timers.

```c++
io_context ioc{1};

timer_context ctx{0ul, max_count, ioc.shutdown_handle()};

// create the timer object
auto timer_obj = ::CreateThreadpoolTimer(
    on_timer_expiration, 
    &ctx, 
    ioc.env());
if (NULL == timer_obj)
{
    throw system_error{};
}
```

```c++
FILETIME due_time{};
timeout_to_filetime(2s, &due_time);

::SetThreadpoolTimer(timer_obj, &due_time, 0, 0);
```

```c++
void __stdcall on_timer_expiration(
    PTP_CALLBACK_INSTANCE, 
    void*     ctx, 
    PTP_TIMER timer)
```

```c++
if (++timer_ctx.count >= timer_ctx.max_count)
{
    ::SetEvent(timer_ctx.shutdown_handle);
}
```

```c++
else
{
    FILETIME due_time{};
    timeout_to_filetime(2s, &due_time);

    ::SetThreadpoolTimer(timer, &due_time, 0, 0);
}
```

### Windows: Waitable Timers and IO Completion Ports

IO completion ports are the standard for asynchronous event multiplexing on Windows. In contrast to interfaces like `epoll()` on Linux which supports a _readiness_ model, IO completion ports expose a _completion_ model. That is, whereas with `epoll()` we use `epoll_wait()` to wait for notification from the kernel that some file descriptor is ready for IO, with IO completion ports we initiate an IO request asynchronously (an operation that completes immediately) and subsequently wait for the operating system to notify us when the request eventually completes.

In order to wait for asynchronous event completions with an IO completion port, one must first register an event source (represented by a Win32 `HANDLE`) with the completion port ...

With Win32 waitable timers, we create and set a timer via calls to `CreateWaitableTimer()` and `SetWaitableTimer()`, respectively. When the timer expires, the timer object itself beomes _signalled_, meaning that we can wait on expiration of a particular timer instance via `WaitForSingleObject()` (or one of its variants). However, this mechanism does nothing for us in the context of integration with IO completion ports. 

Alternatively, one may register a callback function in the call to `SetWaitableTimer()` that will be invoked upon timer expiration. This latter technique sounds like exactly what we need for coroutine integration, but the mechanics of how the callback is invoked pose some issues. Specifically, the operating system does not directly invoke the provided callback function upon expiration of the timer, but instead queues an _Asynchronous Procedure Call_ (APC) to the queue of the thread that made the timer request (called `SetWaitableTimer()`). This mechanism introduces two problems:

1. Standard Windows user APCs do not interrupt the currently executing instruction stream of the thread to which they are queued. Instead, APCs in the APC queue for a thread are only executed in the event that the thread enters an _alertable wait state_ (via a call to e.g. `SleepEx()`).
2. The APC that notifies timer expiration is always queued to the thread that created the timer.

These issues make it difficult to integrate waitable timers with coroutines via IO completion ports.