---
layout: post
title: "Benchmarking Reader/Writer Lock Performance"
---

This post presents some benchmark results for two popular reader-writer lock implementations.

### Motivation: Why Benchmark Reader-Writer Locks?

A project with which I am involved currently uses `tbb::reader_writer_lock` from Intel's Threading Building Blocks (TBB) library to back its internal "shared lock" implementation. However, the TBB library recently marked its `tbb::reader_writer_lock` type for deprecation, suggesting that users replace this type with the standard library's `std::shared_mutex`. Before going ahead with this change, I was curious as to what performance differences (if any) it implied, but direct comparison of the performance characteristics of these two lock implementations proved elusive. Hopefully, this post might serve as a starting point for others looking for such an analysis.  

### Introduction: Reader-Writer Locks

A _Reader-Writer Lock_ is a synchronization primitive that provides mutual exclusion. Like their more popular cousin, the "vanilla" lock, they support the ability to acquire the lock in an exclusive mode - only one thread may hold the lock at a time. However, as their name suggests, reader-writer locks also provide the additional ability to acquire the lock in a shared mode - many threads may hold the lock in shared mode at the same time OR a single thread may hold the lock in exclusive mode. The API for a reader-writer lock `RWLock` might therefore look like:

- `RWLock::lock_shared()` - Acquire the lock in shared mode
- `RWLock::lock_exclusive()` - Acquire the lock in exclusive mode
- `RWLock::unlock_shared()` - Release the lock held in shared mode
- `RWLock::unlock_exclusive()` - Release the lock held in exclusive mode

The API provided by various implementations may differ slightly from this archetype, but the methods above present the general idea.

The benefit of reader-writer locks over vanilla locks is that they provide the ability to increase the level of concurrency supported by an application. In the event that some operations on the data protected by the lock require only read-access, many threads may be allowed to perform these operations simultaneously.

### Reader-Writer Lock Implementations

We'll look at two popular reader-writer lock implementations in this benchmark:

- `tbb::reader_writer_lock`: The reader-writer lock implementation from Intel's TBB library serves threads on a first-come fist-served basis, with the added caveat that writers are given preference over readers - this is often an important modification to ensure that writers are not starved in read-heavy workloads. All waiting is done in userspace via busy-loops. This is a double-edged sword: under low contention, this saves us potentially-expensive calls into the OS kernel, but under high contention this might lead to significant performance degradation as multiple cores are saturated with threads spinning on the lock.
- `std::shared_mutex`: The standard library's reader-writer implementation necessarily differs among compilers and platforms. For this benchmark I use the combination of GCC 10.2 with libstdc++ on Ubunutu 18.04. Under this configuration, `std::shared_mutex` is backed by `pthread_rwlock_t` from the Pthreads library. Unlike `tbb::reader_writer_lock`, `pthread_rwlock_t` uses an _adaptive locking_ approach - the implementation consists of a "fast path" in which waiting threads busy-loop in userspace, eliding a call into the kernel, and a fallback mechanism in which the call into the kernel is performed, allowing threads to wait on the lock without spinning (on Linux this is implemented via the `futex` system call).

The above descriptions should make it evident that the two reader-writer lock implementations differ considerably. But which implementation performs better, and under what conditions?

### Benchmark: Setup

All of the code used to benchmark the performance of these two reader-writer lock implementations is available [on Github](https://github.com/turingcompl33t/cpp-playground/tree/master/locks/rw-bench).

To evaluate the performance of these two reader-writer lock implementations, we will focus on two important aspects of performance: throughput and latency. In this context, _throughput_ refers to the number of lock requests we can process in a given amount of time. In contrast, _latency_ measures the time interval between when a thread requests the lock (in either shared or exclusive mode) and when the request is granted. Both of these performance characteristics are important to the overall performance of a program, but the degree to which this is true depends heavily on the specifics of the application.

### Benchmark: Results

**Throughput**

For throughput evaluation, we benchmark both lock implementations under two workload characteristics:

- _read-heavy_ represents a theoretical workload in which 10% of lock requests are for exclusive access and the remaining 90% of requests are for shared access.
- _equal-load_ represents a theoretical workload in which the number of exclusive and shared locked requests are equivalent.

For each of these workload characteristics, we vary the total number of worker threads running the benchmark (contending on the lock) and measure the total time required for all workers to complete the prescribed number of operations with the lock held. The results are displayed in the plot below.

![Throughput Results](http://raw.githubusercontent.com/turingcompl33t/turingcompl33t.github.io/master/images/2021-1-18-RWLock-Benchmark/throughput.png)

We observe a couple interesting trends here. First, as expected, throughout declines (total benchmark duration increases) between ready-heavy and equal-load workloads - this makes sense because we expect to achieve a higher level of concurrency on the read-heavy workload. The other trend that stands out is the shape of the curve for `std::shared_mutex` under the equal-load workload - we see a massive spike in the time required to complete the benchmark between 20-40 worker threads.

**Latency**

As in the throughput evaluation, we measure latency under both ready-heavy and equal-load workloads. Here, the two workloads are separated into distinct plots because we plot the average reader and writer latency separately, and combining all the data into a single plot becomes unwieldy.

![Read-Heavy Latency](http://raw.githubusercontent.com/turingcompl33t/turingcompl33t.github.io/master/images/2021-1-18-RWLock-Benchmark/latency_read_heavy.png)

Here we may be observing the effect of the writer-preference heuristic in `tbb::reader_writer_lock` as the average latency for readers increases sharply as the number of worker threads increases. In contrast, the latency experienced by readers under `std::shared_mutex` is almost completely static, suggesting that `pthread_rwlock_t` does not include similar provisions for preferring requests from writer threads.

![Equal-Load Latency](http://raw.githubusercontent.com/turingcompl33t/turingcompl33t.github.io/master/images/2021-1-18-RWLock-Benchmark/latency_equal_load.png)

As above, we see that the latency of operations on `std::shared_mutex` is significantly lower than the corresponding latency on `tbb::reader_writer_lock`. Furthermore, we observe the same strange performance behavior that we observed in the throughput experiment for `std::shared_mutex` as the latency experienced by writers increases sharply between 20-40 worker threads before descending to a more reasonable level.

### References / Further Reading

- [API Reference: `std::shared_mutex`](https://en.cppreference.com/w/cpp/thread/shared_mutex)
- [API Reference: `tbb::reader_writer_lock`](https://www.threadingbuildingblocks.org/docs/help/reference/appendices/deprecated_features/considered_for_reworking/reader_writer_lock_cls.html)