---
layout: post
title: "Awaitable System Timers"
---

C++20 coroutines enable many interesting asynchronous programming constructs of their own accord, including `co_await`able synchronization primitives, generator functions, and asynchronous algorithms, to name just a few. However, the power of coroutines really shines when we integrate them with an external event source such as IO completions or timer expirations.

In this post we will explore the latter subject by constructing an `awaitable_timer` type that utilizes the underlying system's native timer facilities to delay resumption of a coroutine until after some delay has elapsed. While this may not seem like a particulary useful primitive on its own, the concepts that we'll cover will be generally applicable to integrating coroutines with any asynchronous system event.

### The Interface

Before we consider the implementation details, let's take a quick look at the interface that we're trying to achieve. With our `awaitable_timer` time, we'll be able to write coroutines such as the one below:

```c++
task<void> async_work()
{
    awaitable_timer two_seconds{2s};

    // do some work

    // suspend this coroutine for two seconds
    co_await two_seconds;

    // do more work after 2 seconds has elapsed
}
```

### Different Systems, Different Timers

Naturally, the timer facilities available to us as application-level developers vary widely based on the operating system on which our program will run. In order to get a feel for how coroutines may be integrated with different OS APIs, I've implemented the `awaitable_timer` construct on three different platforms utilizing each system's native API for asynchronous event management:

- Linux backed by `epoll` and `timerfd`
- OSX backed `kqueue`
- Windows backed by the Windows Threadpool
- Windows backed by IO completion port

As we'll see, the distinct interfaces exposed by each of these system APIs influence the way in which we implement our `awaitable_timer` type.

Without further ado, let's get started.

### Linux: `epoll` and `timerfd`

First up, we'll look at the implementation of `awaitable_timer` on Linux using `epoll` and the `timerfd` API. The code for this portion of the post is located in the [`nix-epoll`](https://github.com/turingcompl33t/coroutines/tree/master/applications/timers/nix-timerfd) directory on Github.

This is not intended to be a post discussing the details or advanced usage of either of these Linux-specific interfaces (there are many great resources on this topic already), but we'll quickly cover the background necessary to understand the timer implementation.

The `epoll` API is Linux's native IO multiplexing interface. It is similar to `poll` in that it allows one to simultaneously monitor many open file descriptors for activity, but it scales much more effectively to large numbers of file descriptors because of some significant differences in their internal implementations.

The core `epoll` API consists of just three function function calls:

- `epoll_create()` or `epoll_create1()`: This function creates a new `epoll` instance and returns a file descriptor that refers to this instance. The original `epoll_create()` accepts a _hint_ that specifies the expected number of file descriptors that will be monitored by this `epoll` instance. However, this hint is ignored by all Linux kernel versions later than 2.6.8. For this reason, the `epoll_create1()` function was introduced; this updated variant no longer accepts a hint and instead allows the caller to specify flags that modify the behavior of the created instance. We will not concern ourselves with the _flags_ argument to `epoll_create1()` in our timer implementation.
- `epoll_ctl()`: This function allows us to manipulate the _interest list_ for a specified `epoll` instance. Supported operations include adding file descriptors to and removing file descriptors from the interest list as well as modifying the context associated with a particular file descriptor on the interest list.
- `epoll_wait()`: This function allows us to wait for readiness events on any of the file descriptors on the interest list for a particular `epoll` instance. We pass an array of `epoll_event` structures and upon return these structures are populated with information for each ready file descriptor, including the event that made the file descriptor ready (e.g. readable, writeable) and the context that was associated with the descriptor at the time the descriptor was added to the interest list.

Next up, the basics of `timerfd`. The `timerfd` API allows us to create system timers that are represented by file descriptors. The only two `timerfd` API functions that we need to know for this post are the following:

- `timerfd_create()` This function creates a new timer instance and returns a file descriptor that refers to this timer. At the the time of creation, a timer is not yet set to expire.
- `timerfd_settime()`: This function arms a timer created by a previous call to `timerfd_create()` such that it will expire at some point in the future. The API allows for specification of the expiration time in either absolute or relative terms, but we will only consider relative timeouts in this post. Furthermore, the `timerfd_settime()` function also allows us to specify whether we would like the timer to be periodic or expire only once. Because our ultimate goal is to construct a timer type upon which we can `co_await` and such an operation is intended to be a one-shot delay of execution, we are not interested in periodic timers for the purposes of our timer implementation.

The `timerfd` API integrates beautifully with `epoll` because it allows us to the register each timer in which we are interested with the `epoll` instance via the standard `epoll_ctl()` call. Whenever a timer expires, the timer becomes _ready_ and a thread currently monitoring for readiness events via `epoll_wait()` will be woken up. 

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

Now that we have the context for the timer specified, we can register the timer with the `epoll` instance. When a `timerfd` timer expires, the file descriptor is considered _readable_, so when adding the timer to the `epoll` interest list we specify that we are interested in _readable_-readiness on this file descriptor with the `EPOLLIN` flag.

```c++
struct epoll_event ev;
ev.events   = EPOLLIN;
ev.data.ptr = &ctx;

if (::epoll_ctl(instance.get(), EPOLL_CTL_ADD, timer.get(), &ev) == -1)
{
    throw system_error{};
}
```

The final two things that we do in the body of `main()` are arm the timer to expire in 2 seconds and begin running the reactor loop. `arm_timer()` is really just a wrapper around `timerfd_settime()` that arms the timer to expire once (i.e non-periodically) after the specified delay.

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

Now that we understand the basics of setting and waiting for system timers with `epoll` and `timerfd`, how do we integrate this coroutines to create and `awaitable_timer`?

The implementation for our `awaitable_timer` type is provided in the [`awaitable_timer.hpp`](https://github.com/turingcompl33t/coroutines/blob/master/applications/timers/nix-timerfd/awaitable_timer.hpp) header. 

The constructor for `awaitable_timer` accepts the file descriptor for the `epoll` instance that will monitor for timer expirations and the delay after which this timer should fire. In the body of the constructor we simply create a new timer via `timerfd_create()` and subsequently associate the file descriptor for this timer with with `epoll` instance via `epoll_ctl()`. Notice that at this point that we do not arm the timer because we need to defer this operation until the point at which we `co_await` upon the `awaitable_timer` instance.

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

When we register the timer with the `epoll` instance we specify a pointer to the `awaitable_timer`'s `async_ctx` member as the context associated with the file descriptor. Here, `async_ctx` is simply a wrapper around the `coroutine_handle<>` for the coroutine that will eventually be resumed upon timer expiration. We achieve this by specifying `on_timer_expire()` as the callback function that the thread running the reactor should invoke when a timer expires and providing the `async_ctx` pointer as an argument to this callback (via an opaque `void*` pointer) which we can then use to resume the waiting coroutine.

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

Next up is an implementation of `awaitable_timer` on OSX backed by `kqueue`. The code for this portion of the post is located in the [osx-kqueue](https://github.com/turingcompl33t/coroutines/tree/master/applications/timers/osx-kqueue) directory on Github.

`kqueue` is the native event-notification system on several operating systems including OSX and many BSD variants. Like `epoll` on Linux, `kqueue` is a readiness-based system in which we (application programmers) register interest in kernel events and are then able to wait on notification from the kernel when these events occur. However, `kqueue` presents a slightly more general interface by introducing the notion of a `kevent` - a tuple of a unique identifier, a _filter_ that describes the events of interest for the identifier, and (optionally) some user-provided context to associate with the identifier. Internally, the `kqueue` system uses the filter for each `kevent` to determine if the `kevent` is _ready_ and, if it is, inserts the `kevent` onto the `kqueue` so that application code can later retrieve it.

The core API for `kqueue` consists of just two system calls:

- `kqueue()`: This call constructs a new `kqueue` instance and returns a file descriptor that refers to this instance. 
- `kevent()` This call is used to both register new `kevent`s with the `kqueue` instance as well as query any notified `kevent`s that are already associated with the instance. In a way, the `kevent` system call functions as a combination of both `epoll_ctl()` and `epoll_wait()`.

Another major difference between the `epoll` API and the `kqueue` API is that `kqueue` has "built-in" support for timers in the form of a timer filter. That is, whereas with `epoll` we had to use the `timerfd` API to construct a timer, associate this timer with the `epoll` instance, and subsequently wait for readiness of the timer via a call to `epoll_wait()`, with `kqueue` we need only specify the timer filter (`EVFILT_TIMER`) in a call to `kevent()` to add a new timer `kevent` to the kernel-maintained `kqueue` instance. Note that if you search for the OSX `kqueue` API (e.g. via Google) the top result from the Apple developer manual pages erroneously states that the `EVFILT_TIMER` filter is not supported on OSX. 

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

The default behavior with the `EVFILT_TIMER` filter is to establish a periodic timer. This implies that the code above registers a `kevent` that will be inserted into the `kqueue` every `us.count()` microseconds until we disable it manually.

Finally, we use the `kevent()` system call once more to wait for notification that the timer `kevent` has been added to the `kqueue`.

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

Two down, one to go.

### Windows: The Windows Threadpool

There are a number of viable ways to create waitable system timers on Windows, and these different mechanisms significantly influence the approach that we take in making these timers compatible with coroutines.

The code for this portion of the post is located in the [win-pool](https://github.com/turingcompl33t/coroutines/tree/master/applications/timers/win-pool) directory on Github.

The most obvious method for creating timers on Windows is via the [waitable timer interface](https://docs.microsoft.com/en-us/windows/win32/sync/waitable-timer-objects). However, some issues arise when attempting to integrate Windows waitable timers with the IO completion port event multiplexing interface. It is possible to address these issues and utilize this approach successfully to build awaitable system timers, but it requires a significantly more involved implementation. We'll take a look at this approach in the subsequent section. For now, we'll pursue a simpler approach that utilizes the Windows threadpool.

The Windows threadpool API allows one to submit various types of requests to a pre-allocated pool of worker threads managed by the operating system. The API is [extensive](https://docs.microsoft.com/en-us/windows/win32/procthread/thread-pool-api), giving application-level programmers plenty of flexibility in working with the threadpool. Furthermore, because the threadpool is implemented by the operating system and even has some components that run in the kernel, it is able to take advantage of information regarding the threadpool's current utilization to dynamically tune itself for optimal performance - something that is not possible in a user-space threadpool implementation. See [this excellent video](https://channel9.msdn.com/Shows/Going+Deep/Inside-Windows-8-Pedro-Teixeira-Thread-pool) from Channel9 for more information regarding some of these optimizations that were introduced with Windows 8.

Among the operations supported by the Windows threadpool API is the submission of timer requests. One creates a new threadpool timer object (not to be confused with a waitable timer object) with `CreateThreadpoolTimer()` and subsequently submits this timer to the threadpool with `SetThreadpoolTimer()`. We specify a callback during timer creation, and when the timeout specified in the call to `SetThreadpoolTimer()` expires, a threadpool worker thread executes the provided callback.

With that introduction out of the way, let's dive into integrating coroutines with the threadpool API. In the code samples that follow, I introduce an `io_context` type that wraps the Windows threadpool API. The definition of this type is unimportant with respect to coroutine integration with system timers; the class merely serves to encapsulate some (somewhat convoluted) logic that allows us to create a process-local threadpool rather than utilize the default system-wide threadpool. The results would be effectively identical without this additional abstraction.

The core of the implementation appears in the [`awaitable_timer.hpp`](https://github.com/turingcompl33t/coroutines/blob/master/applications/timers/win-pool/awaitable_timer.hpp) header which defines the `awaitable_timer` type. The constructor for the type accepts a reference to the `io_context` instance that represents the threadpool to which this timer instance should submit timer requests as well as the timeout for this timer instance.

```c++
template <typename Duration>
awaitable_timer(io_context& ioc_, Duration timeout_)
    : ioc{ioc_}, timeout{}, timer{}, async_ctx{}
{
    // convert the specified timeout
    timeout_to_filetime(timeout_, &timeout);

    // create the underlying timer object
    auto* timer_obj = ::CreateThreadpoolTimer(
        on_timer_expiration, 
        &async_ctx, 
        ioc.env());

    if (NULL == timer_obj)
    {
        throw coro::win::system_error{};
    }

    timer.reset(timer_obj);
}
```

When the underlying timer object is created in the constructor of `awaitable_timer`, we specify `on_timer_expiration()` as the callback to be invoked on expiration of this timer and we specify that a pointer to our `async_ctx` member be provided as the optional context argument to this callback.

Our `async_ctx` member is nothing more than a wrapper around a coroutine handle; it need not be a distinct type at all, but this approach does allow for simple refactors in the event that we decide that we want to pass additional context to the callback in future updates to our implementation. We will see later that this coroutine handle is set in the `await_suspend()` member of the `awaiter` type for `awaitable_timer` and thus represents a handle to the coroutine that `co_await`s on the timer's expiration.

```c++
struct async_context
{
    stdcoro::coroutine_handle<> awaiting_coro;
};
```

The `on_timer_expiration()` callback is a static member of the `awaitable_timer` type. Its signature is dictated by the Win32 `PTP_TIMER_CALLBACK` type and it receives both a pointer to the callback instance as well as a pointer to the timer object that triggered its invocation when it is called. However, for our purposes, we only care about the optional user-provided context that is provided to `on_timer_expiration()` in the form of an opaque pointer (`void*`). The only action that we take when the timer expires is extract the coroutine handle for the awaiting coroutine from the asynchronous context passed to the callback and subsequently resume the coroutine:

```c++
static void __stdcall on_timer_expiration(
    PTP_CALLBACK_INSTANCE, 
    void* ctx, 
    PTP_TIMER)
{
    auto& async_ctx = *reinterpret_cast<async_context*>(ctx);
    async_ctx.awaiting_coro.resume();
}
```

So we've seen how the coroutine that `co_await`s the timer is resumed, but how does the coroutine submit the timer and suspend itself in the first place? The `operator co_await()` member of `awaitable_timer` defers this work to the `timer_awaiter` type:

```c++
timer_awaiter awaitable_timer::operator co_await()
{
    return timer_awaiter{*this};
}
```

As `awaiter`s go, `timer_awaiter` is not particularly complicated. As we saw above, it accepts a reference to the `awaitable_timer` instance with which it is associated in its constructor, which it subsequently utilizes in its `await_suspend()` implementation in order to set the awaiting coroutine handle for the `awaitable_timer` instance. Once it has accomplished this, `await_suspend()` submits the timer request to the threadpool with `SetThreadpoolTimer()` and unconditionally suspends the awaiting coroutine.

```c++
class timer_awaiter
{
    awaitable_timer& timer;
public:
    timer_awaiter(awaitable_timer& timer_)
        : timer{timer_}
    {}

    bool await_ready()
    {
        return false;
    }

    void await_suspend(std::experimental::coroutine_handle<> awaiting_coro)
    {
        timer.async_ctx.awaiting_coro = awaiting_coro;
        ::SetThreadpoolTimer(timer.timer.get(), &timer.timeout, 0, 0);
    }

    void await_resume() {}
};
```

Usage of this `awaitable_timer` implementation is analogous to the previous usages we have seen - we simply construct a new timer instance that is associated with our `io_context` threadpool wrapper and subsequently `co_await` the timer.

```c++
awaitable_timer two_seconds{ioc, 2s};

for (auto i = 0ul; i < n_reps; ++i)
{   
    co_await two_seconds;

    puts("[+] timer fired");
}
```

And that's all there is to it! The high-level nature of the Windows threadpool API makes integration with coroutines incredibly simple. However, if you compare this implementation with the `epoll` and `kqueue` examples we saw previously, we did lose something here: fine-grained control over the threads that drive execution of our coroutines. After successfully `co_await`ing on an `awaitable_timer` instance, the body of the invoking coroutine will thereafter be executed by a thread that is managed by the Windows threadpool. This might be fine in some instances, and indeed ensures that system resources are utilized efficiently, but it lacks the "bring your own threads" level of control that was available in the `epoll` and `kqueue` models wherein we, the application programmer, were responsible for providing the threads that drove the reactor and ultimately resumed any suspended coroutines.

It is possible to regain this level of control on Windows, but doing so obviously necessitates that we ditch the threadpool API and migrate to managing timer expiration events with an IO completion port.

### Windows: Waitable Timers and IO Completion Ports

IO completion ports are the standard for asynchronous event multiplexing on Windows. In contrast to interfaces like `epoll()` on Linux and `kqueue` on OSX which expose a _readiness_ model, IO completion ports expose a _completion_ model. That is, whereas with `epoll` we use `epoll_wait()` to wait for notification from the kernel that some file descriptor is ready for IO, with IO completion ports we initiate an IO request asynchronously (an operation that completes immediately) and subsequently wait for the operating system to notify us when the request eventually completes.

The fundamentals of the IO completion port API consists of the following operations:

- Create a new IO completion port with `CreateIoCompletionPort()`; this function returns a handle to a new IO completion port object.
- Register a handle with an existing IO completion port with `CreateIoCompletionPort()` (yes, the same function does both; poor API design in my opinion). The handle we register with the completion port is the handle for which we want to receive notifications on IO event completions. Many different handle types are supported, including regular (filesystem) files, sockets, pipes, etc. After registration with the completion port, we will receive notification of IO completion events on the handle we register via `GetQueuedCompletionStatus()` (see below).
- Manually post completion events with `PostQueuedCompletionStatus()`. In contrast to events queued to the completion port via registered handles, this function allows us to manually post a completion event to the IO completion port. This functionality comes in handy for special-case events such as the handling of manual shutdown requests for server applications backed by a completion port.
- Wait for completion events with `GetQueuedCompletionStatus()`. This function is typically invoked in a loop in order to repeatedly wait for a completion event and dispatch the corresponding action based on the event properties.

Now that we understand the basics of IO completion port interface, let's turn our attention to Win32 waitable timers and begin considering how we might integrate them with a completion port-backed reactor. 

With Win32 waitable timers, we create and set a timer via calls to `CreateWaitableTimer()` and `SetWaitableTimer()`, respectively. When the timer expires, the timer object becomes _signalled_, meaning that we can wait on expiration of a particular timer instance via `WaitForSingleObject()` or one of its variants. However, this mechanism does nothing for us in the context of integration with IO completion ports because it does not wake a thread blocked on `GetQueuedCompletionStatus()`. 

Alternatively, one may register a callback function in the call to `SetWaitableTimer()` that will be invoked upon timer expiration. This latter technique sounds like exactly what we need for coroutine integration, but the mechanics of how the callback is invoked pose some issues. Specifically, the operating system does not directly invoke the provided callback function upon expiration of the timer; instead, it queues an _Asynchronous Procedure Call_ (APC) to the queue of the thread that made the timer request (i.e. the one that called `SetWaitableTimer()`). This introduces two issues:

1. Standard Windows user APCs do not interrupt the currently executing instruction stream of the thread to which they are queued. This is a great feature in my opinion because it avoids all of the complexity that arises when you introduce truly-asynchronous OS-level event notification like you get with POSIX signals. Windows takes a different approach in which APCs in the APC queue for a thread are only executed in the event that the thread enters an _alertable wait state_. There are various Win32 function calls that cause a thread to enter such a state, including `SleepEx()` and `WaitForSingleObjectEx()`. On its own, the APC model is not a dealbreaker for integration with IO completion ports because the extended variant of `GetQueuedCompletionStatus()`, `GetQueuedCompletionStatusEx()`, supports alertable waits. 
2. The APC that notifies timer expiration is always queued to the thread that created the timer. This implies that whichever thread calls `SetWaitableTimer()` in order to submit a timer request must also be the one that invokes the alertable wait function (`SleepEx()`, `GetQueuedCompletionStatusEx()`, etc.) that allows the thread to be notified on timer expiration.

This second point is the one that necessitates a more involved approach to waitable timer integration with IO completion ports. We want to achieve the same behavior that we implemented in the `epoll` and `kqueue` examples wherein we suspend a coroutine by `co_await`ing on an `awaitable_timer` instance and move on, assured that this coroutine will be resumed by one of the threads running the IO service. Because of the limitation that the waitable timer callback is always queued to the thread that submitted the timer, we can't support this behavior with the naive approach.

As a workaround, we'll introduce a dedicated timer management thread. Instead of creating and setting their own waitable timers, our coroutines will instead send timer requests to the timer management thread which will subsequently submit these requests to the operating system on the coroutine's behalf. When timers subsequently expire, the timer management thread will be notified via an APC and will in turn post a completion event to an IO completion port that is ultimately responsible for resuming our coroutines.

The source in [win-iocp](https://github.com/turingcompl33t/coroutines/tree/master/applications/timers/win-iocp) implements this approach. The code is significantly more involved than the previous examples so I won't cover the entirety of the implementation here, but I will highlight the interesting parts.

The `timer_service` class encapsulates both the IO completion port that is capable of multiplexing arbitrary IO event completions as well as the auxiliary worker thread that is dedicated to managing timer submissions and expiration events.

The constructor for the `timer_service` launches the timer worker thread which runs the `process_requests()` function as its entry point. In this function, the worker thread enters a loop where it blocks on a queue `pop()` operation and submits a timer request to the operating system whenever it successfully dequeues a request from the queue.

```c++
unsigned long __stdcall timer_service::process_requests(void* ctx)
{
    // retrieve the timer service context
    auto& me = *reinterpret_cast<timer_service*>(ctx);

    for (;;)
    {
        auto* req = me.requests.pop(TRUE);

        ::SetWaitableTimer(
            req->timer_object, 
            reinterpret_cast<LARGE_INTEGER*>(&req->timeout), 
            0,
            on_timer_expiration,
            req,
            FALSE);
    }
}
```

Hidden behind this call to `requests.pop()` is the fact that the worker thread enters an alterable wait state while it waits on a nonempty queue. It is this alertable wait state that allows the timer worker thread to be notified by APCs when the timers that it submits eventually expire.

```c++
T* pop(BOOL alertable)
{
    ::WaitForSingleObjectEx(lock, INFINITE, alertable);
    while (is_empty())
    {
        ::SignalObjectAndWait(lock, non_empty, INFINITE, alertable);
        ::WaitForSingleObjectEx(lock, INFINITE, alertable);
    }

    // ...
}
```

A coroutine submits a timer to the timer service by `co_await`ing on one of the `timer_service` members: `timer_service::post_awaitable()` or `timer_service::post_awaitable_with_handle()`. In the `await_suspend()` member of the returned `timer_service::awaitable`, we create a new timer request and push it onto the queue of timer requests.

```c++
std::unique_ptr<timer_request> request = /* ... */

// ...

service.requests.push(request, FALSE);
```

As we saw above, the timer worker thread sits on the other end of this queue, constantly attempting to dequeue and submit new requests to the operating system. Notice here that we always pass the timer request to the timer worker thread for submission to the operating system so that it can subsequently be notified via APC when the timer expired; if instead an arbitrary thread executing a coroutine submitted the request to the operating system, the timer worker thread would not receive this notification.

When a timer expires, the timer worker thread is notified via an APC and it invokes `on_timer_expiration()` callback function. In this function, the timer worker simply posts the timer completion event to the IO completion port so that it can be handled via the regular event dispatching logic.

```c++
void timer_service::on_timer_expiration(
    void* ctx, 
    unsigned long /* dwTimerLowValue  */, 
    unsigned long /* dwTimerHighValue */)
{
    auto* req = reinterpret_cast<timer_request*>(ctx);

    ::PostQueuedCompletionStatus(
        req->port, 0, 0, reinterpret_cast<LPOVERLAPPED>(req));
}
```

To complete the process, when a timer completion event is posted to the IO completion port, one of the threads that is currently executing `timer_service::run()` receives the notification that the timer expired via `GetQueuedCompletionStatus()` and subsequently invokes the callback function that is embedded in the request which, for requests initiated via a `co_await` on one of `timer_service::post_awaitable()` or `timer_service::post_awaitable_with_handle()`, consists solely of resuming the suspended coroutine. 

Before wrapping up, there is one limitation to this implementation that I want to address: while it functions correctly, this approach is inefficient in terms of operating system resource usage. We create a new OS-level timer object for every request made to the `timer_service`. In these trivial example programs, this is not an issue, but in larger applications that might see hundreds or thousands of concurrent outstanding timer requests, this might become a performance bottleneck. One way to address this limitation is to manage our own queue of outstanding timer requests at the application level, and only ever have a single timer object that is managed at the operating system level for the timer with the earliest due time. This is the approach taken in more robust IO service implementations, such as the one provided by [Boost.Asio](https://think-async.com/Asio/) and that provided in the [CppCoro](https://github.com/lewissbaker/cppcoro/blob/master/lib/io_service.cpp) library.

### Concluding Thoughts

We covered a lot of ground in this post. My hope is that it provides a broad overview of the various asynchronous event mulitplexing APIs provided by different operating systems and the way in which the distinctions between these APIs influence how we integrate coroutines with operating system services. Understanding the details of implementing this integration lays a solid foundation for developing coroutine support for other IO interfaces such as pipes and network sockets. 