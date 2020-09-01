---
layout: post
title: "Increasing Map Multi-Lookup Throughput with Coroutines"
---

In this post we'll explore an approach to using C++20 coroutines to hide memory stall latency when performing bulk lookup operations on a map data structure that exceeds the size of our last-level cache.

All of the code associated with this post is available on [Github](https://github.com/turingcompl33t/coroutines/tree/master/applications/map).

### Acknowledgements

My interest in this topic was sparked by Gor Nishanov's presentation at CppCon 2018: [Nano-Coroutines to the Rescue!](https://www.youtube.com/watch?v=j9tlJAqMV7U). In that presentation, Gor walks through the implementation of a binary search algorithm that utilizes coroutines to hide memory stall latency. Furthermore, in the [paper](http://www.vldb.org/pvldb/vol11/p1702-jonathan.pdf) on which Gor's presentation is based, the authors utilize a chaining hashtable implementation as another case study for the same approach to performance improvement. My implementation here is an attempt to recreate the results reported in that paper.

### The Setting

Map (or _associative array_) interfaces are ubiquitous in many programming environments. We use maps to count the number of occurrences of unique words in a text corpus. We use maps to maintain dynamic configuration information in large systems. We use maps to implement software caching solutions. In short, maps are everywhere. 

However, one similarity among all the examples presented above is that we are typically only interested in single-lookup operations on the map. That is, at any one time, we only want to look up a single key in the map, either to merely test for its presence or to retrieve the associated value. 

One particularly salient example of this type of usage is in the implementation of _hash join_ algorithms for database management systems. A full discussion of the mechanics of a hash join is outside the scope of this post (see e.g. any of the slides / lectures available from [CMU's database system's courses](https://www.cs.cmu.edu/~pavlo/) for a more robust explanation) but I'll cover the absolute basics here so that we have a feel for a potential use case.

Briefly, in a simple two-way hash join, we construct a hashtable that is keyed by the join attribute from the first table, inserting each key into the table. Then, to perform the join, we iterate over each join attribute in the second table and probe the hashtable for this value. If the attribute is present in the table (the lookup succeeds) then the tuple that corresponds to that join attribute should be included in the output of the join.

### The Problem

When the number of keys in the map grows large, the underlying storage utilized by the map to maintain its items may exceed the size of L3 cache, implying that some lookup 

This problem is particularly salient in pointer-based data structures wherein there exist one or more levels of pointer-based indirection between the "entry" to the data structure and the item in which a lookup operation is interested. The classic chaining hashtable implementation of the map API is an example of one such data structure. 

### Approach

The general approach we'll take to hiding the latency from memory stalls associated with cache misses is called _instruction stream interleaving_, or ISI. As the name implies, under ISI we maintain and multiplex multiple, independent instruction streams concurrently, interleaving them intelligently to keep the CPU busy with computation rather than stuck waiting for the memory system. The high-level approach proceeds in two steps:

1. Spawn multiple independent instruction streams in which we issue a hardware prefetch instruction that directs the memory system to pull the value(s) (more accurately, the cache lines on which they reside) upon which the computation represented by the stream depends from main memory into one of the CPU caches.
2. Once all streams are started and the prefetches are issued, return to the first stream (the one that was initiated first) and proceed with computation. At this point, the prefetch that was issued previously has (hopefully) completed, and the the value(s) that we utilize in the computation will not incur a cache miss.

C++20 coroutines are a perfect match for implementing ISI because they provide us with an elegant and lightweight abstraction for maintaining multiple instruction streams and their associated context. The latter property is particularly important for the application we have in mind; if the cost of switching between executing coroutines is higher than the cost of a memory stall from an L3 cache miss, any work we do to interleave instruction streams will be for naught as we will be assured to see an overall decline in performance for our troubles.

As mentioned before, the entirety of this implementation is available [on Github](https://github.com/turingcompl33t/coroutines/tree/master/applications/map) so I won't go through all of the details here. However, we will look at two short code snippets that get to the heart of how we implement ISI with coroutines.

The first code snippet we'll look at is the coroutine that represents a single lookup task. Accordingly, this coroutine is implemented by a member function `Map::lookup_task()`, the body of which is shown below.

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

If you compare this function with the implementation of regular (non-interleaved) lookup operations on the map (`Map::lookup()`) you'll find that the two are remarkable similar. We replace the `return` statements with `co_return`, and instead of returning `LookupKVResult` directly, we invoke one of two callback functions that are provided to the coroutine as parameters (this just simplifies the implementation of the function that manages these coroutines, which we'll see next). The only other difference are the `co_await prefetch_and_schedule_on()` expressions that take the place of raw pointer dereferences. The `prefetch_and_schedule_on()` function issues a hardware prefetch on the provided address, schedules the coroutine that invoked it for resumption on the provided `Scheduler`, and suspends execution of the coroutine. When the coroutine is subsequently resumed by the `Scheduler`, the `co_await prefetch_and_schedule_on()` expression simply resolves to the desired pointer via the definition of the `await_resume()` member of the `prefetch_awaiter` that `prefetch_and_schedule_on()` returns.

The only other code we'll look at is the `Map::interleaved_multilookup()` function itself. 

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

### Results

The general approach we'll use for benchmarking the two multilookup implementations is the following:

- Construct a `Map` instance with a specified number of key / value pairs 
- Perform a multilookup operation on the entire set of key / value pairs in the `Map` and record the time it takes to complete the entirety of the operation
- Compute the cost of a single lookup operation by dividing the total duration of the multilookup by the number of keys in the `Map`

The hardware statistics for the system on which these benchmarks were run are as follows:

- 6 X 1700 MHz CPUs (CPU count is immaterial, the benchmark is single-threaded)
- 32 KiB L1 Data Cache
- 1,024 KiB L2 Unified Cache
- 8,448 KiB L3 Unified Cache

All of the following statistics utilize a `Map` instance with a maximum capacity of 65,536 (`2^16`).

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

For the results that follow, 10 streams are utilized for interleaved multilookup. In initial experiments I also varied this parameter over the range 8-32 streams, but saw very little difference in the performance achieved.

| Input Size (Keys) | Sequential Multilookup | Interleaved Multilookup |
|------------------:|-----------------------:|------------------------:|
|            65,536 |                 125 ns |                  393 ns |
|           131,072 |                 129 ns |                  431 ns |
|           262,144 |                 141 ns |                  498 ns |
|           524,288 |                 182 ns |                  628 ns |
|         1,048,576 |                 284 ns |                  871 ns |
|         2,097,152 |                 942 ns |                1,323 ns |
|         4,194,304 |                2425 ns |                2,206 ns |
|         8,388,608 |                6737 ns |                3,944 ns |
|        16,777,216 |              11,414 ns |                7,460 ns |
|        33,554,432 |              26,988 ns |               15,891 ns |

### Further Work

I only evaluated the performance of interleaved multilookup using the implementation of coroutines provided by GCC 10. It would be interesting to compare the performance of the benchmark across the implementation provided by the other major compilers. Prior work in this area seems to suggest that the coroutines implementation in Clang is the most mature among the three major compilers, so we might see an even greater performance benefit there.

### References / Further Reading

- [Exploiting Coroutines to Attack the "Killer Nanoseconds"](http://www.vldb.org/pvldb/vol11/p1702-jonathan.pdf)
- [Interleaving with Coroutines: A Practical Approach for Robust Index Joins](http://www.vldb.org/pvldb/vol11/p230-psaropoulos.pdf)
- [Gor Nishanov, CppCon 2018: Nano-Coroutines to the Rescue!](https://www.youtube.com/watch?v=j9tlJAqMV7U)