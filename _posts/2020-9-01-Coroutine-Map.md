---
layout: post
title: "Accelerating Map Multi-Lookup with Coroutines"
---

In this post we'll explore an approach to using C++20 coroutines to hide memory stall latency when performing bulk lookup operations on a data structure that exceeds the size of last-level cache.

All of the code associated with this post is available on [Github](https://github.com/turingcompl33t/coroutines/tree/master/applications/map).

### Acknowledgements

My interest in this topic was sparked by Gor Nishanov's presentation at CppCon 2018: [Nano-Coroutines to the Rescue!](https://www.youtube.com/watch?v=j9tlJAqMV7U). In that presentation, Gor walks through the implementation of a binary search algorithm that utilizes coroutines to hide memory stall latency. Furthermore, in the [paper](http://www.vldb.org/pvldb/vol11/p1702-jonathan.pdf) on which Gor's presentation is based, the authors utilize a chaining hashtable implementation as another case study for the same approach to performance improvement. My implementation here is an attempt to recreate the results reported in that paper.

### A Motivating Example

Suppose we are building an application and a requirement arises for a data structure that implements the "map" interface; that is, one that support lookups, insertions, updates, and removals on key / value pairs of arbitrary type. Having taken Computer Science 101 as undergraduates, and blithely unaware of the [many](https://abseil.io/about/design/swisstables) [alternative](https://github.com/efficient/libcuckoo) [designs](https://github.com/skarupke/flat_hash_map) available, we settle on a chaining hashtable for the underlying implementation of our `Map` type.

Further, assume one additional, slightly-irregular requirement: periodically, higher-level logic in our application requests a lookup operation on a large subset of the keys stored in our `Map` all at once, and is unable to make progress until the entirety of this bulk lookup operation completes, requiring that our `Map` support efficient multilookup. The API for this function might resemble the following:

```c++
template <
    typename BeginInputIter, 
    typename EndInputIter, 
    typename OutputIter>
auto sequential_multilookup(
    BeginInputIter begin_keys,
    EndInputIter   end_keys,
    OutputIter     begin_results) -> void;
```

When the number of keys stored in our `Map` is small, life is good. The entirety of the data structure fits within one of the levels of our cache hierarchy, and all of the lookups performed as part of a multilookup operation are served from cache. Consequently, we observe high throughput, as measured by the number of completed lookups per second, for the operation as a whole.

```c++
Map<int, int> our_map{};

// insert a moderate number of key / value pairs
our_map.insert(0, 1);

// ...

// perform a multilookup
our_map.sequential_multilookup(
    search.begin(), 
    search.end(), 
    std::back_inserter(results));

// go on doing other important computations
```

However, once the number of keys stored in the map grows large enough, and the data structure exceeds the size of our last-level cache, performance problems emerge. The overall throughput of the multilookup operation falls off a cliff once this threshold is reached, and continues to grow worse in a superlinear fashion as the map's size increases further. Naturally we expect lower throughput as the number of items in the map increases because of the additional cost associated with traversing deeper into bucket chains that grow in length with each hash collision, but the performance hit is far worse than our expectations. What is happening?

```c++
our_map.sequential_multilookup(
    search.begin(), 
    search.end(), 
    std::back_inserter(results));

// ... blocked
// ... blocked
// ... blocked
```

For regular memory access patterns, hardware prefetching logic built into our processor is able to accurately predict future memory accesses and fetch memory regions from main memory such that they are located in one of our CPU caches prior to the access actually occurring, ensuring that the eventual read or write can be serviced quickly. However, in the case of our chaining hashtable, the memory access pattern is far from regular - indeed we have inadvertently created a data structure and associated algorithm (multilookup) that exhibits an almost-pathologically bad memory access pattern. The linked-lists that we use to manage hash collisions are the first problem; each entry in the linked-list for a single bucket has no relationship with the entries before or after it in the list in terms of their physical proximity in the address space of our program - they might be located megabytes apart, and only refer to eachother via the pointers they store. If the entries in a given linked-list were allocated sequentially, this might not be such a common occurrence as the underlying allocator that actually performs our memory allocations will likely service temporally-proximate requests for the same size block of memory from a relatively small region. However, our implementation actively works against this scenario by way of our hash function - a computation whose sole purpose is to uniformly distribute keys across the entire range of buckets that compose our table. 

Now that we understand the memory access pattern implied by our hashtable design, the reasons underlying the sharp performance degradation we observe become obvious. As we probe the hashtable in a multilookup operation, we spend most of our time traversing the linked-lists that manage our hash collisions. Whenever we fail to find the current search key at the current entry, we attempt to dereference the `next` pointer in the current entry in order to move on to the next one. However, when the total memory footprint of the hashtable far exceeds the size of our last-level cache, there is a high probability that this results in a cache miss, and the hardware prefetcher is unable to do anything to help us. Thus, instead of doing useful work like comparing the key stored in the current entry with the search key, the CPU enters a memory stall in which it is stuck waiting for the memory system to service the cache miss - an incredibly high-latency operation relative to the typical processing speed of modern CPUs.

As a brief aside, the performance issues that our `Map` is experiencing are not unique to the chaining hashtable implementation that we selected; any data structure that introduces pointer-based indirection, such as one of the tree variants, would see similar results, although likely not to the same degree as we experience with our naive design.

Alright, now that we understand the issues inherent in our implementation, what can we do to address them? 

### Coroutines and Instruction Stream Interleaving

The general approach we'll take to hiding the latency from memory stalls associated with cache misses is called _instruction stream interleaving_, or ISI. As the name implies, under ISI we maintain and multiplex multiple, independent instruction streams concurrently, interleaving them intelligently to keep the CPU busy with computation rather than stuck waiting for the memory system. The high-level approach proceeds in two steps:

1. Spawn multiple independent instruction streams in which we issue a software prefetch instruction that directs the memory system to pull the value(s) (more accurately, the cache lines on which they reside) upon which the computation represented by the stream depends from main memory into one of our CPU caches.
2. Once all streams are started and the prefetches are issued, return to the first stream (the one that was initiated first) and proceed with computation. At this point, the prefetch that was issued previously has (hopefully) completed, and the the value(s) that we utilize in the computation will not incur a cache miss.

C++20 coroutines are a perfect match for implementing ISI because they provide us with an elegant and lightweight abstraction for maintaining multiple instruction streams and their associated context. The latter property is particularly important for the application we have in mind; if the cost of switching between executing coroutines is higher than the cost of a memory stall from a last-level cache miss, any work we do to interleave instruction streams will be for naught and we will instead see an overall decline in performance.

To implement multilookup with coroutines-enabled ISI, we add a new member function to our `Map` implementation with the following signature:

```c++
template <
    typename BeginInputIter, 
    typename EndInputIter, 
    typename OutputIter,
    typename Scheduler>
auto interleaved_multilookup(
    BeginInputIter    begin_keys,
    EndInputIter      end_keys,
    OutputIter        begin_results,
    Scheduler const&  scheduler,
    std::size_t const n_streams) -> void;
```

The first three arguments for this function specify the input and output ranges utilized by the multilookup operation, exactly as in the `sequential_multilookup()` member from before. However, the `interleaved_multilookup()` function accepts two additional arguments, the first of which is a reference to a `Scheduler` that manages suspension and subsequent resumption of the individual coroutines it spawns, and the second that specifies the number of independent instruction streams to utilize in the multilookup. We'll revisit these two parameters in slightly more detail below.

### Behind the Scenes

As mentioned before, the entirety of this implementation is available [on Github](https://github.com/turingcompl33t/coroutines/tree/master/applications/map) so we won't go through all of the details here. However, we will look at two short code snippets that get to the heart of how we leverage C++20 coroutines to implement instruction stream interleaving.

The first snippet we'll look at is the body of `Map::interleaved_multilookup()` itself. Within this function, we instantiate a `Throttler` instance that manages the individual coroutines we'll use to manage distinct instruction streams. The `Throttler` type is really just a thin wrapper around the user-provided `Scheduler` instance that effectively limits the maximum number of concurrently-active coroutines to the value specified by `n_streams`. With the `Throttler` instantiated, we simply iterate over the entire range of search keys, spawning a `lookup_task()` coroutine for each one. When the `Throttler` attempts to `spawn()` a new coroutine and finds that the maximum number of active streams has been reached, it asks the `Scheduler` for the next available coroutine that it is managing and resumes it.

```c++
using MapType    = Map<KeyT, ValueT, Hasher>;
using ResultType = typename MapType::LookupKVResult;

Throttler throttler{scheduler, n_streams};

for (auto key_iter = begin_keys; key_iter != end_keys; ++key_iter)
{
    throttler.spawn(
        lookup_task(
            *key_iter,
            scheduler,
            [&begin_results](KeyT const& k, ValueT& v) mutable {
                *begin_results = ResultType{k, v};
                ++begin_results;
            },
            [&begin_results]() mutable { 
                *begin_results = ResultType{};
                ++begin_results;
            }));
}

// run until all lookup tasks complete
throttler.run();
```

Accordingly, the next piece of code we'll look at is the `Map::lookup_task()` member function. This is the coroutine that represents a single lookup task and is thus responsible for searching the `Map` for the single key that it is provided as an argument.

```c++
auto const index = bucket_index_for_key(key);

auto& bucket = buckets[index];
if (0 == bucket.n_items)
{
    co_return on_not_found();
}

auto* entry = co_await prefetch_and_schedule_on(bucket.first, scheduler);

for (;;)
{
    if (key == entry->key)
    {
        co_return on_found(entry->key, entry->value);
    }

    if (nullptr == entry->next)
    {
        // reached the end of the bucket chain
        break;
    }

    // traverse the linked-list of entries for this bucket
    entry = co_await prefetch_and_schedule_on(entry->next, scheduler);
}

co_return on_not_found();
```

If you compare this function with the implementation of regular (non-interleaved) lookup operations on the map (`Map::lookup()`) you'll find that the two are remarkably similar. We replace the `return` statements with `co_return`, and instead of returning `LookupKVResult` directly, we invoke one of two callback functions that are provided to the coroutine as parameters. This approach just simplifies the implementation of `Map::interleaved_multilookup()` because we don't have to deal with the return values of each of our coroutine tasks. The only other differences are the `co_await prefetch_and_schedule_on()` expressions that take the place of raw pointer dereferences. The `prefetch_and_schedule_on()` function issues a software prefetch on the provided address, schedules the coroutine that invoked it for resumption on the provided `Scheduler`, and suspends execution of the coroutine. When the coroutine is subsequently resumed by the `Scheduler` (via the `Throttler`), the `co_await prefetch_and_schedule_on()` expression resolves to the desired pointer via the definition of the `await_resume()` member of the `prefetch_awaiter` that `prefetch_and_schedule_on()` returns.

The body of `Map::lookup_task()` illustrates the tremendous advantage that a coroutine-based approach to instruction-stream interleaving has over alternatives: the code is nearly identical to the synchronous version and is therefore relatively easy to reason about. For this simple example, this is not such a big deal, but in larger applications the development cost of implementing a more complex approach may prove prohibitive. For a point of comparison, see [this paper](http://www.vldb.org/pvldb/vol9/p252-kocberber.pdf) that describes one such alternative, dubbed _asynchronous memory access chaining_, and compare the implementation difficulty inherent in the two approaches.

### Experimental Results

So this solution to our performance woes sounds great in theory, but does it actually work? To determine the effectiveness of our couroutine-based multilookup with support for instruction stream interleaving, we'll benchmark both the naive `sequential_multilookup()` implementation with our enhanced version `interleaved_multilookup()`.

The general approach we'll take in benchmarking the two multilookup implementations is the following:

- Construct a `Map` instance with a specified number of key / value pairs 
- Perform a multilookup operation on the entire set of key / value pairs in the `Map` and record the time it takes to complete the entirety of the operation
- Compute the cost of a single lookup operation by dividing the total duration of the multilookup by the number of keys in the `Map`, as well as the keys-per-second throughput that the entire operation achieved

The hardware statistics for the system on which the following benchmarks were run are as follows:

- 6 X 1700 MHz CPUs (CPU count is immaterial, the benchmark is single-threaded)
- 32 KiB L1 Data Cache
- 1,024 KiB L2 Unified Cache
- 8,448 KiB L3 Unified Cache

We'll test our two multilookup operations on multiple `Map` instances, each with a different number of key / value pairs inserted so we can see how the performance characteristics of each algorithm evolve as the total memory footprint of the `Map` changes. The size of the `Map` instances used in the benchmark are displayed in the table below. All of the following statistics utilize a `Map` instance with a maximum capacity of 65,536, meaning that the number of buckets allocated to the table does not grow beyond this value. The _average bucket depth_ assumes a uniform distribution of keys, which we realize in the benchmark by using integer keys that increase monotonically from 0 up to the input size.

| Input Size (Keys) | Memory Footprint | Average Bucket Depth |
|------------------:|-----------------:|---------------------:|
|            65,536 |             1 MB |                    1 |
|           131,072 |             2 MB |                    2 |
|           262,144 |             4 MB |                    4 |
|           524,288 |             8 MB |                    8 |
|         1,048,576 |            16 MB |                   16 |
|         2,097,152 |            32 MB |                   32 |
|         4,194,304 |            64 MB |                   64 |
|         8,388,608 |           128 MB |                  128 |
|        16,777,216 |           256 MB |                  256 |
|        33,554,432 |           512 MB |                  512 |
{: .my-class }

Finally, we arrive at the results of the benchmark. In the results that follow, **SM** is an abbreviation for _sequential mulitlookup_ while **IM** is an abbreviation for _interleaved multilookup_. The throughput achieved by each algorithm is expressed in millions of keys per second (M Keys/Sec). Ten instruction streams are utilized for `Map::interleaved_multilookup()`; in initial experiments I also varied this parameter over the range 8-32 streams, but saw very little difference in resulting performance.

| Input Size (Keys) | **SM** ns/Key | **SM** M Keys/Sec | **IM** ns/Key | **IM** M Keys/Sec |
|-----------:|---------:|---------------:|---------:|-----------------:|
|     65,536 |   125 ns |              8 |   393 ns |             2.54 |
|    131,072 |   129 ns |           7.75 |   431 ns |             2.30 |
|    262,144 |   141 ns |           7.09 |   498 ns |             2.00 |
|    524,288 |   182 ns |           5.49 |   628 ns |             1.59 |
|  1,048,576 |   284 ns |           3.52 |   871 ns |             1.15 |
|  2,097,152 |   942 ns |           1.06 | 1,323 ns |             0.76 |
|  4,194,304 |  2425 ns |           0.41 | 2,206 ns |             0.45 |
|  8,388,608 |  6737 ns |           0.15 | 3,944 ns |             0.25 |
| 16,777,216 |11,414 ns |          0.088 | 7,460 ns |             0.13 |
| 33,554,432 |26,988 ns |          0.037 |15,891 ns |            0.062 |

There are a couple of trends to notice here. The first is that for small `Map` instances with a memory footprint that fits entirely within last-level cache, `Map::sequential_multilookup()` crushes `Map::interleaved_multilookup()`. This should not be surprising: in these cases, the software prefetching and coroutine multiplexing performed by the interleaved algorithm bring no performance benefits to the algorithm and still incur the overhead associated with these additional operations. 

However, as the size of the `Map` increases, we begin to see the payoff from our interleaved algorithm as the time per operation rises much more slowly than in the sequential algorithm and consequently the observed throughput remains higher. For the largest `Map` instance tested, a 512 MB data structure that far exceeds the capacity of last-level cache, the throughput of `Map::interleaved_multilookup()` is almost 70% higher than its sequential counterpart.

### Further Work

I only evaluated the performance of interleaved multilookup using the implementation of coroutines provided by GCC 10. It would be interesting to compare the performance of the benchmark across the implementations provided by the other major compilers. Prior work in this area seems to suggest that the coroutines implementation in Clang is the most mature among the three major compilers, so we might see an even greater performance benefit there.

More importantly, the chaining hashtable implementation we explored in this post is little more than a simple proof-of-concept; you would likely be hard-pressed to find a `Map` implemented this way in production because so many alternative implementations exist that are known to be more efficient. It wil be interesting to experiment with applying this technique to higher-performing data structures to determine if we might achieve benefits similar to those we observed here, although perhaps not to the same degree.

### References / Further Reading

- [Exploiting Coroutines to Attack the "Killer Nanoseconds"](http://www.vldb.org/pvldb/vol11/p1702-jonathan.pdf)
- [Interleaving with Coroutines: A Practical Approach for Robust Index Joins](http://www.vldb.org/pvldb/vol11/p230-psaropoulos.pdf)
- [Asynchronous Memory Access Chaining](http://www.vldb.org/pvldb/vol9/p252-kocberber.pdf)
- [Gor Nishanov, CppCon 2018: Nano-Coroutines to the Rescue!](https://www.youtube.com/watch?v=j9tlJAqMV7U)