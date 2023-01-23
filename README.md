# Notes for the design of a concurrent hashmap API

## Prior art

### oneTBB

[`oneapi::tbb::concurrent_hash_map`](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_hash_map_cls.html)
introduces the notion of [_accessors_](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_hash_map_cls/accessors.html),
objects with smart pointer semantics
that provide locked access to a given element. For instance, the following:
```cpp
tbb_map_type::accessor acc;
map.emplace(acc, key, 0);
++acc->second;
```
emplaces a new element (or locates an equivalent one) and provides access to it via `acc`. There
are const (shared lock) and non-const (exclusive lock) accessors. Member functions
`find`, `insert`, `emplace` have additional overloads accepting accessors. Rather than returning
iterators, these functions return a `bool` (indicating if the element was found in the case
of `find`, or whether the element has been inserted in the case of `insert` and `emplace`).

Besides `find`, `insert` and `emplace`, additional functionality is provided _without
concurrency guarantees_:
* iterators,
* assignment,
* `clear` and `swap`.

Parallel iteration is served through [_container ranges_](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/named_requirements/containers/container_range.html),
which behave like C++17 ranges but provide additional splitting functionalities to
distribute the range load traversal across threads.
