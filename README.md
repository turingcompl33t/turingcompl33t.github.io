### Post Ideas

This section of the README describes some ideas for potential future posts.

**What Has My Standard Library Done for Me Lately?**

I think it might be fun and interesting to look at some of the more advanced aspects of the C++ standard library in a series of posts. The goal would not be to (necessarily) re-implement the standard library types and algorithms but rather to explore some of the optimizations the authors of the standard library use "under the covers" to make the standard library implementations more efficient than the naive solution that developers might come up with on their first pass at the problem in question. Some preliminary post ideas might include:

- `std::string`: small string optimization
- `std::function`: small function optimization
- `std::vector`: resize / allocation strategy? such a popular type really anything might be of interest here
- `std::shared_ptr`: allocation strategy with the constructor vs with `std::make_shared`
- `std::mutex`: fast path (spinning in user space, at least when backed by pthreads)

### Acknowledgements

Original layout / boilerplate for this site cloned from the [jekyll-now](https://github.com/barryclark/jekyll-now) project.