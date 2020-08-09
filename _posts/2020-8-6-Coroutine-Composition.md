---
layout: post
title: "Coroutine Composition with Tasks"
---

This blog post explores the semantics of coroutine composition by examining a simplified implementation of the `task` type.

All of the code for this post is available [on Github](https://github.com/turingcompl33t/coroutines). I successfully built and ran the examples used here under both GCC (`g++ 10.1`) on Ubunutu and Clang (`clang++ 11.0`) on OSX.

### Disclaimer

As suggested in the above summary, the implementation of `task` explored in this post lacks a number of key features that would be necessary for it to see use in non-trivial code. First and foremost, our `task` does not support returning values upon task completion. I made the decision to omit return values from this `task` implementation because I find that the necessity of dealing with generics (via templates) in the implementation distracts from the primary objective of the post which is to understand coroutine composition. 

Another feature missing from this implementation of `task` is thread safety - it is _not safe_ to utilize this `task` type in a multithreaded program. We will look at the potential races inherent in this limited implementation later on in the post.

For a more complete, production-quality `task` implementation, see the [`task.hpp`](https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/task.hpp) header in the [cppcoro library](https://github.com/lewissbaker/cppcoro) library on Github.

### Introduction: What is a `task`?

In general, asynchronous programming terms, a _task_ is merely a unit of work. For our purposes here, we'll be slightly more specific and define a task as a lazily-evaluated asynchronous computation. The _asynchronous_ portion of this definition implies that the evaluation of a task may take place whilst other work is being performed in the system and that we may have many tasks active simultaneously while the _lazy_ nature of a task implies that the computation required by the task is not started until we are certain that its value will be required by some higher-level component in the system.

You would be correct in thinking that this definition sounds an awful lot like certain asynchronous programming primitives already available in C++, namely `std::future`. We can launch a task with `std::async()` which returns a `std::future` for the result of the task (alternatively, we could launch the task manually and subsequently obtain the `std::future` from a `std::promise` we create via `std::promise::get_future()`). However, there are a [number](https://www.youtube.com/watch?v=QIHy8pXbneI) [of](https://eli.thegreenplace.net/2016/the-promises-and-challenges-of-stdasync-task-based-parallelism-in-c11/) [reasons](https://bartoszmilewski.com/2009/03/03/broken-promises-c0x-futures/) that `std::future` fails to meet the requirements necessary for a robust task-based parallelism abstraction for C++., namely the obvious performance concerns and the current inability to easily compose them. The most important thing to understand about tasks is that distinct tasks do not necessarily imply distinct _execution contexts_, the most familiar of which would be an operating system thread. This implies that many tasks are capable of sharing the same execution context which (often) greatly reduces the overhead of creating new tasks and destroying them when they complete. 

The introduction of coroutines allow us to make significant progress towards true task-based parallelism, and the `task` type we implement here serves as a fundamental building block in the larger space of parallel programming abstractions.

### Basic Usage of the `task` Type

Before exploring the nature of coroutine composition, let's first ensure that we understand the basics of how the `task` type behaves. The header `task_v0.hpp` contains the definition of a barebones `task` type. We'll start at the top-level `task` type and work our way down through both `task_promise` and `task_awaiter` to ensure that we understand all of the moving pieces.

The constructor and destructor for the `task` type are unsurprising. A `task` is an RAII wrapper for the coroutine to which it is attached, so we contruct a `task` with the coroutine handle to the coroutine to which it refers and upon destruction of the `task` instance we destroy the associated coroutine frame.

```c++
task(coro_handle_type coro_handle_)
    : coro_handle{coro_handle_} {}

~task()
{   
    if (coro_handle)
    {
        coro_handle.destroy();
    }
}
```

Next up we see that `task` defines `operator co_await` that returns a `task_awaiter`:

```c++
task_awaiter operator co_await();
```

The body of `operator co_await` for task is defined later, after the definition of the `task_awaiter` type. All the operator does is construct and return a new `task_awaiter` from the coroutine handle owned by the `task`.

```
task_awaiter task::operator co_await()
{
    return task_awaiter{this->coro_handle};
}
```

Finally, `task` implements a `resume()` member function:

```c++
bool resume()
{   
    if (!coro_handle.done())
    {
        coro_handle.resume();
    }

    return !coro_handle.done();
}
```

This function is only here so that we can manually resume the root `task` in our example program. A production `task` implementation would likely not implement such a method because, in general, we don't want to be manually driving our `task`s; instead, we would rely upon some other event source, such as an IO reactor servicing network requests to resume suspended `task`s and drive them toward completion.

Next up is the `task_awaiter` type. In the case of `task_awaiter`, the three member functions required to implement the `awaiter` interface are trivial:

```c++
bool await_ready()
{
    return false;
}

void await_suspend(coro_handle_type awaiting_coro_)
{   
    coro_handle.resume();
}

void await_resume() {}
```

The `task_awaiter` unconditionally reports that it is not ready in its `await_ready()` member. In `await_suspend()`, we unconditionally resume the coroutine to which this `awaiter` is attached, and subsequently yield control back to the caller (indicated by the `void` return type of `await_suspend()`). In `await_resume()` we are given the opportunity to determine the final result of the `co_await` expression that created this `task_awaiter`, but our `task` type does not return a value, so it does not make sense to return anything here, hence we do nothing.

The final piece of the puzzle is the `task_promise` type which determines the properties of any coroutine that returns our `task`.

The `get_return_object()` member simply constructs a new `task` from a coroutine handle to the associated coroutine:

```c++
task get_return_object()
{
    return task{coro_handle_type::from_promise(*this)};
}
```

We return `suspend_always{}` from our `initial_suspend()` member in order to ensure that our `task` type is lazy - the body of the coroutine that returns a `task` does not begin executing until the returned `task` is `co_await`ed upon by the caller. If instead we returned `suspend_never{}`, our `task` would be launched as soon as we called the coroutine that returns a `task` instance.

```c++
auto initial_suspend()
{
    return stdcoro::suspend_always{};
}
```

Our `final_suspend()` also returns `suspend_always{}` to ensure that our `task` is a proper RAII wrapper; we want the coroutine frame for the coroutine associated with a `task` to remain "alive" until the `task` has been destroyed. If we instead specified `suspend_never{}` here, the coroutine frame would be destroyed immediately after execution of the body of the coroutine completed and we would no longer have control over the lifetime of the coroutine frame.

```c++
auto final_suspend()
{
    return stdcoro::suspend_always{};
}
```

The final two members of the `task_promise` type are deeply uninteresting. Our `return_void()` is a no-op, and we punt on error handling and simply terminate the program in the event that an exception propagates out of the coroutine body.

```c++
void return_void() {}

void unhandled_exception() noexcept
{
    std::terminate();
}
```

This is all that is required to implement a simple `task` type that enables lazy, asynchronous computation. 

Now that we've walked through the internals of the type, let's take it for a spin. The `example0.cpp` program employs our `task` type in a very straightforward setting. We define a single coroutine `async_task()` that returns a `task` object. In the body of the `async_task()` coroutine we simply suspend execution of the coroutine once by `co_await`ing on `suspend_always{}`. 

```c++
task async_task()
{
    co_await stdcoro::suspend_always{};
}
```

The body of `main()` simply constructs a new `task` and subsequently invokes `task::resume()` in order resume its execution.

```c++
auto t = async_task();
t.resume();
t.resume();
```

Notice that we need to invoke `task::resume()` twice in order to drive the coroutine to completion; because the `task` type is lazy, execution of the body of `async_task()` does not begin until the first time that we `resume()` the `task`. The first time around, execution reaches the `co_await suspend_always{}` statement and returns to the caller. When we resume the coroutine once more by invoking `resume()` on its `task`, we pick up immediately after the `co_await` on `suspend_always{}` and subsequently fall off the end of the coroutine body.

The code in `example0.cpp` includes some `trace()` statements that simply print out a message along with the function from which they are called. Executing `example0` produces the following output which confirms our theoretical understanding of how the `task` type should behave:

```
[main] enter
[main] after call to async_task()
[async_task] enter
[main] after task::resume() #1
[async_task] exit
[main] after task::resume #2
[main] exit
```

### Composing Coroutines with `task`: A First Try

Alright enough talk about the `task` type itself; let's see how we can use `task` to compose coroutines and create nested asynchronous operations.

The program `example1.cpp` contains the setup for this experiment. Using the same `task` type that we implemented previously, it sets up a chain of nested coroutines.

```c++
task completes_synchronously()
{
    co_return;
}

task coro_2()
{
    co_await completes_synchronously();
}

task coro_1()
{
    co_await coro_2();
}

task coro_0()
{
    co_await coro_1();
}
```

The `main()` function behaves exactly as before, creating a `task` by invoking `coro_0()` and invoking `task::resume()` on the returned `task` until the `task` reports that it is complete:

```c++
int main()
{   
    auto t = coro_0();
    while (t.resume());
}
```

If we run the `example1` program, the following output is produced:

```
[coro_0] enter
[coro_1] enter
[coro_2] enter
[completes_synchronously] enter
[completes_synchronously] exit
[coro_0] exit
```

If you're anything like me, this result might surprise you the first time you see it. What happened here? All three of our coroutines began execution of their bodies as we would expect, and the `completes_synchronously()` coroutine runs to completion synchronously (as the name implies), but after this we only see the statement that acknowledges resumption of the coroutine `coro_0()` and the `coro_1()` and `coro_2()` coroutines are never resumed and thus never complete. 

What one might _expect_ to see (or perhaps _want_ to see) is resumption of `coro_2()` after `completes_synchronously()` completes, followed by `coro_1()`, and finally `coro_0()` - as if our coroutines formed a structure that was built up and subsequently torn down according to the same rules. But this expectation is a symptom of a misunderstanding of the way coroutines work, and to see this we need only more closely consider the execution flow of our example program.

The traversal "down" the coroutine chain is uninteresting and proceeds exactly as one would expect. At the bottom of the stack, `coro_2()` waits on the result of `completes_synchronously()` with `co_await` to begin execution of this synchronous task. It is important to realize here, however, that the statement `co_await completes_synchronously()` has two implications for `coro_2()`:

1. It suspends execution of `coro_2()` and
2. It transfers control to `completes_synchronously()`

Now, when `completes_synchronously()` subsequently completes and immediately returns to its caller, `coro_2()` _is still suspended_ and thus cannot resume its execution even though the `task` upon which it `co_await`ed is complete. In this way, control flow propagates all the way back up to `main()` where the second "issue" arises. In `main()`, we only have access to the `task` created by the call to `coro_0()`, and when we invoke `task::resume()` on this `task`, naturally it is `coro_0()` that is resumed. 

We might think that it would make sense at this point for `coro_0()` to resume `coro_1()` (and so on down to `coro_2()`) because the `co_await` in these coroutines has also been satisfied, but `coro_0()` has no _responsibility_ and indeed not even an _ability_ to resume the `task` upon which it `co_await`ed - all `coro_0()` "knows" is that the wait was satisfied, nothing more about the state of execution of `coro_1()`.

This situation should help to develop an intuition for an extremely important concept when programming with C++ coroutines: when a coroutine suspends, it is up to that coroutine to schedule itself for resumption at some point in the future. In our current implementation, all three of our coroutines suspend their execution with a `co_await`, but they do not take any steps to ensure that they will later be resumed; the only reason that we see `coro_0` complete is the top-level "external" resumption of this coroutine by the call to `task::resume()` in `main()`, but this facility does nothing for the nested coroutines `coro_1()` and `coro_2()`. 

The missing piece of our implementation emerges from the preceding paragraphs; what we need to fix our `task` type is some way by which a `task`-returning coroutine may immediately resume execution of its caller upon completion. This way, when a `co_await` on a `task` is satisfied, the coroutine that is performing the `co_await` may immediately be resumed by the `task` once it completes.

### Composing Coroutines with `task`: Adding Continuations

The addition that we need is called a _continuation_. Continuations are another general concept in asynchronous programming that allow one to express the idea: "do Y once X completes." A more familiar example of a continuation is the `.then()` method of a `future` (as yet still missing from the standard library implementation but available in other libraries e.g. `boost::future`) which allows one to chain the next computation that should be performed with the result of the `future` instead of blocking with a call to `.get()`. 

With coroutines, adding continuations to the `task` type means that we need to implement some mechanism by which `task` B, after being launched by a `co_await` in `task` A, may subsequently resume execution of the coroutine associated with `task` A once it completes.

The code changes required to implement continuations for our `task` type are relatively minor and all take place within the `task_promise` and `task_awaiter` types.

First, the `task_awaiter`. The only change here is in the `await_suspend()` member:

```c++
void await_suspend(stdcoro::coroutine_handle<> awaiting_coro)
{
    coro_handle.promise().continuation = awaiting_coro;
    coro_handle.resume()
}
```

Here, `coro_handle` is the coroutine handle with which the `task_awaiter` was constructed, which means it is the coroutine handle for the `task` that is `co_await`ed upon. In contrast, `awaiting_coro` is the coroutine handle for the coroutine in which `co_await <expr that produces task>` is invoked. The specification of the `await_suspend()` member of the `awaiter` interface tells us that this handle is always passed to the `co_await`ed upon coroutine in this way. We then take this handle to the coroutine that is suspending itself and store it in the promise object for the coroutine to which control is now being transferred.

The updates to `task_promise` complete the picture. All of the changes are in the `final_suspend()` member:

```c++
auto final_suspend()
{
    struct final_awaiter
    {
        task_promise& me;

        bool await_ready()
        {
            return false;
        }

        void await_suspend(coro_handle_type)
        {
            if (me.continuation)
            {
                me.continuation.resume();
            }
        }

        void await_resume() {}
    };

    return final_awaiter{*this};
}
```

Whereas in our previous implementation `final_suspend()` returned `suspend_always{}` to unconditionally suspend itself at the coroutine's _final suspend label_, we now return a `final_awaiter` type that implements the `awaiter` interface. Now when a `task`-returning coroutine completes and the compiler implicitly inserts the call to `co_await p.await_suspend()` (where `p` is the notional `promise` object for the coroutine), the `await_suspend()` member of the `final_awaiter` is invoked and, if a continuation has been registered by way of another coroutine `co_await`ing on the current one, the awaiting coroutine is immediately resumed via the coroutine handle stored in the promise.

The updated implementation of the `task` type is included in the `task_v1.hpp` header. The `example2.cpp` program is identical to `example1.cpp` apart from the fact that it utilizes the updated `task` implementation. Executing `example2` demonstrates the desired behavior for our `task`:

```
[main] enter
[coro_0] enter
[coro_1] enter
[coro_2] enter
[completes_synchronously] enter
[completes_synchronously] exit
[coro_2] exit
[coro_1] exit
[coro_0] exit
[main] exit
```

### Concluding Thoughts

In this post we explored the basics of coroutine composition by developing a `task` type that supports continuations. While simple and not sufficiently robust to be used in production code, our `task` type demonstrates many of the concepts that one must understand and internalize in order to write programs that make effective use of asynchrony and coroutines.