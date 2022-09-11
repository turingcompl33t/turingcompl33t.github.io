---
layout: post
title: "What has my Standard Library Done for me Lately? - the boolean specialization of std::vector"
---

This short post explores the `std::vector<bool>` specialization of `std::vector` in the C++ standard library. Specifically, we look at how the standard library writers achieve a space-optimized implementation without sacrificing (most) of the semantics we expect from standard containers, and why this optimization sometimes leads to unexpected behavior.

### Motivation

The boolean specialization of `std::vector` (aka `std::vector<bool>`) has received criticism from the C++ community for years. I will leave the debate over the utility of the container to [more experienced developers](https://isocpp.org/blog/2012/11/on-vectorbool), but I would still like to understand the source of the "weirdness" associated with `std::vector<bool>`. To that end, we'll first look at some examples of `std::vector<bool>`'s misbehavior before diving into the source to determine where it originates.

In his classic [Effective STL](https://www.amazon.com/Effective-STL-Specific-Standard-Template/dp/0201749629), Scott Meyers advises readers to avoid use of `std::vector<bool>` because it is not a proper STL container. He demonstrates this idea with a simple example:

```c++
std::vector<int> v{1, 2, 3};
int* i = &v[0];
```

This is perfectly valid C++. However, by replacing `int` with `bool`, we produce code that no longer compiles:

```c++
std::vector<bool> v{true, true, false};
bool* i = &v[0];
```

In this instance, the compiler will report that we cannot assign the result of `&v[0]` to `bool*` because the types are incompatible. With this example, Meyers demonstrates that `std::vector<bool>` does not support some of the semantics we expect from an STL container - namely we cannot get the location of an element (its address) in a familiar way.

As a more realistic example, suppose that we want to write our own version of the STL's [`std::for_each`](https://en.cppreference.com/w/cpp/algorithm/for_each). Naturally, this implementation must be generic over the type of the element in the container. If we constrain the container type to be `std::vector`, our implementation might resemble the following:

```c++
template <typename T, typename F>
auto for_each(std::vector<T> const& container, F func) -> void {
  for (auto& item : container) {
    func(item);
  }
}
```

This algorithm works fine for nearly all types for the contained elements. For `int`, it works as expected:

```c++
auto main() -> int {
  std::vector<int> container{1, 2, 3};
  for_each(container, [](int const& i) { printf("%d\n", i); });
}
```

Produces:

```bash
1
2
3
```

However, this code breaks when we attempt to pass a `std::vector<bool>` to our algorithm. The version of `main()` below

```c++
auto main() -> int {
  std::vector<bool> container{true, true, false};
  for_each(container, [](bool const& i){ printf("%d\n", i); });
}
```

produces the following error message (on GCC x86-64 12.2):

```
<source>: In instantiation of 'void for_each(const std::vector<T>&, F) [with T = bool; F = main()::<lambda(const int&)>]':
<source>:13:13:   required from here
<source>:6:5: error: cannot bind non-const lvalue reference of type 'bool&' to an rvalue of type 'std::_Bit_const_iterator::const_reference' {aka 'bool'}
    6 |     for (auto& item : container) {
      |     ^~~
Compiler returned: 1
```

Again we see that the error stems from the fact that the range-`for` construct expects a `bool&` type, but the container's iterator is failing to provide this type and is instead returning a `std::_Bit_const_iterator::const_reference` type. Next we'll look at the implementation of a simplified version of `std::vector<bool>` to understand why this type-mismatch occurs.

### A Naive `std::vector<bool>`

Before working on our implementation of `std::vector<bool>`, let us first consider why this specialization is provided in the first place. Why not just let `bool` follow the same rules as all of the other types?

With most compilers and standard library implementations, the size of a single `bool` object is 1 byte (8 bits). With GCC, the code

```c++
printf("%zu\n", sizeof(bool));
```

prints the value `1`. Furthermore, in the implementation of `std::vector`, each item in the container is allocated its own storage area in a contiguous memory region (subject to alignment constraints). Together, these two facts imply that for a non-specialized implementation of `std::vector<bool>`, **each boolean element takes 1 byte**.

It is easy to see that such an implementation is space-efficient. Each boolean element consumes a full byte of space, when we know that **the information contained in a `bool` can be expressed by a single bit**.

However, there is no primitive type in C++ that has a storage requirement that is smaller than a single byte. For this reason, C++ standard authors advocate a clever implementation that introduces some "hacks" to compress the space allocated to each element of the `std::vector` down to a single bit.

### A More Efficient Implementation

To demonstrate a boolean `std::vector` specialization in the spirit of those offered by the C++ standard library, we'll write our own `BitVector` type. The complete implementation is [here](https://github.com/turingcompl33t/cpp-playground/blob/master/containers/bvector/BitVector.hpp). Note that this implementation is considerably stripped-down and does not offer the full interface provided by `std::vector`. furthermore, it is implemented in a style that would never be utilized within the standard library (for instance, it uses `std::unique_ptr` internally). This is by design, as I believe it demonstrates the topics of interest without getting bogged-down in the complexities of a more mature implementation.

Our key implementation strategy for space-efficiency is representing many `bool` items in the vector with a single element in the underlying storage area. We manage individual bits of a primitive integer type and provide some logic on top to encode and decode `bool` values from these bits.

We begin by declaring a `BitType` type alias that defines the type that we will use in the implementation of our underlying storage array - in this case, `unsigned long` (the same used by GCC / libstdc++). We then declare an array of these elements owned by a `std::unique_ptr` to automatically handle deallocation.

```c++
using BitType = unsigned long;

class BitVector {
  /** The number of elements in the vector */
  std::size_t size_;
  /** The total capacity of the vector */
  std::size_t capacity_;
  /** The underlying storage */
  std::unique_ptr<BitType[]> storage_;
...
}
```

The `At()` member function demonstrates how we read boolean values from our underlying storage:

```c++
auto At(std::size_t index) const -> bool {
  if (index >= size_) {
    throw std::out_of_range("Index out of range.");
  }
  ASSERT(index < size_, "Broken precondition.");
  const auto mask = 1UL << index % sizeof(BitType);
  return storage_[index / sizeof(BitType)] & mask;
}
```

First, we select the "bucket" (really, the word) in which the bit of interest is stored. Then, we construct a mask that isolates the bit of interest and apply this to the word to read a `bool` value.

The `PushBack()` member works similarly to write new values to our `BitVector`:

```c++
auto PushBack(bool const val) -> void {
  if (size_ == capacity_) {
    Expand();
  }
  ASSERT(capacity_ > size_, "Broken invariant.");
  if (val) {
    // The 'bucket' in which new bit is set
    auto const bucket = size_ / sizeof(BitType);
    storage_[bucket] |= (1UL << size_ % sizeof(BitType));
  }
  size_++;
}
```

Initially, we note that we do not need to update the underlying storage at all in the event that a `false` value is inserted - only the metadata member `size_` needs to be updated. In the event that `true` is inserted into the `BitVector`, we follow the same procedure as before to isolate the bit of interest, and we write to the word in which it resides using the bitwise or (`|=`) operator.

Now we have seen how we can read and write boolean values to our `BitVector` while only allocating each a single bit of storage space. The final piece of the puzzle is the mechanism for returning (possibly mutable) references to individual bits in our container. As noted earlier in this post, C++ offers us no mechanism by which we can address a location that consists of a single bit. This complicates the implementation of certain member functions, like `operator[]`. Consider what the prototype for this function might look like:

```c++
auto operator[](std::size_t index) -> ??? {
  # ???
}
```

We have to provide a way to read and write an individual bit in the container without the ability to directly return a reference to it. The way that the C++ standard library implementors solve this problem is through the use of _reference proxies_.

### Reference Proxies

Our final implementation of `operator[]` ends up looking like:

```c++
auto operator[](std::size_t index) -> BitRef {
  if (index >= size_) {
    throw std::out_of_range("Index out of range.");
  }
  ASSERT(index < size_, "Broken precondition.");
  auto* bucket = &storage_[index / sizeof(BitType)];
  const auto mask = 1UL << index % sizeof(BitType);
  return BitRef{bucket, mask};
}
```

The magic of this implementation lies in the `BitRef` type which implements a reference proxy for the individual bits in our `BitVector`. It consists of a pointer to the word that it references in the `BitVector` (`BitType*`) combined with the mask that isolates the bit of interest within the word:

```c++
class BitRef {
  /** The 'bucket' in which the bit is located */
  BitType* const bucket_;
  /** The mask to isolate the target bit */
  BitType const mask_;

  BitRef(BitType* bucket, BitType const mask)
      : bucket_{bucket}
      , mask_{mask} {}
}
```

The `operator bool()` member function demonstrates how `BitRef` "reaches back" to its originating `BitVector` instance to read the boolean value that it represents:

```c++
operator bool() const { return !!(*bucket_ & mask_); }
```

Likewise, `operator=(bool)` demonstrates how we can use the `BitRef` to assign to an individual bit in the originating `BitVector`:

```c++
auto operator=(bool const val) -> BitRef& {
  if (val) {
    *bucket_ |= mask_;
  } else {
    *bucket_ &= ~mask_;
  }
  return *this;
}
```

These two member functions illustrate the core functionality of the `BitRef` proxy type. See the [complete implementation](https://github.com/turingcompl33t/cpp-playground/blob/master/containers/bvectors/BitVector.hpp) for more details.

### Conclusion

This implementation only demonstrates the use of reference proxies to aid the implementation of member functions like `operator[]`, `Front()`, and `Back()`. A distinct but related technique, that of an _iterator proxy_, is employed by the standard library to allow `std::vector<bool>` to be used with constructs like the range-for loop.

Hopefully this quick walkthrough demystifies the errors that arise when working with `std::vector<bool>`. We see that the core of the issue stems from the fact that the specialization introduces a space-efficient implementation that allocates a single bit for each `bool` element. Because we cannot return a reference to an individual bit, we introduce a proxy object. These proxies provide the illusion of a reference to a single bit in the container but the illusion breaks down under some conditions, and this can make working with this type perilous.

### References

- [C++ Weekly 325: Why vector of bool is Weird](https://www.youtube.com/watch?v=OP9IDIeicZE)
- [On vector of bool](https://isocpp.org/blog/2012/11/on-vectorbool)
