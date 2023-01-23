# Notes for the design of a concurrent hashmap API

## Prior art

### oneTBB

[`oneapi::tbb::concurrent_hash_map`](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_hash_map_cls.html)
introduces the notion of [_accessors_](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_hash_map_cls/accessors.html),
objects with smart pointer semantics that provide locked access to a given element.
For instance, the following:
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
There are non-standard overloads of `insert` accepting a key rather than a full value (so,
`insert(acc, k)` behaves approximately as `try_emplace(acc, k, mapped_type())`).
`operator[]` is not provided. `erase([const_]iterator)` is logically replaced by
`erase([const_]accessor)`.

Besides `find`, `insert`, `emplace` and `erase`, additional functionality is provided _without
concurrency guarantees_:
* iterators,
* assignment,
* `clear` and `swap`.

Parallel iteration is served through [_container ranges_](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/named_requirements/containers/container_range.html),
which behave like C++17 ranges but provide additional splitting functionalities to
distribute the range load traversal across threads. Parallel iteration is
not concurrency-safe, that is, when parallel-iterating a container no other
operations can be performed on the container.

### libcuckoo

[`libcuckoo::cuckoohash_map`](https://efficient.github.io/libcuckoo/classlibcuckoo_1_1cuckoohash__map.html)
does not have iterators. The concurrency-safe lookup/modify interface is as follows:
* `mapped_type find(const K&)`, `bool find(const Key&, mapped_type&)`, `bool find_fn(const K&, F)`
* `bool contains(const K&)`
* `bool insert(K&&, Args&&...)`, `bool insert_or_assign(K&&, V&&)`
* `bool update_fn(const K&, F)`
* `bool uprase_fn(K&&, F, Args&&...)`
* `bool upsert(K&&, F, Args&&...)`
* `bool erase(const K&)`

`mapped_type find(const K&)` throws if the element is not found. `bool find(const Key&, mapped_type& x)`
copies the mapped value of the element found to `x`. Despite its name, `bool insert(K&&, Args&&...)`
has rather the semantics of `std::unordered_map::try_emplace`. `update_fn` is equivalent to
`find_fn`, except that the user-provided function is passed a non-const reference to the
mapped value. `uprase_fn` behaves like `std::unordered_map::try_emplace`, but additionally
passes a reference to the mapped value to the user-provided function, along with information on
whether the element has been newly created or was preexistent: the function can further indicate
if the elements is to be erased. `upsert` is a variation of `uprase_fn` where the element
is never erased. `operator[]` is not provided.

Assignment is not concurrency-safe. `rehash`, `reserve` and `clear`, on the other hand,
can be called in a concurrent scenario.

`lock_table` blocks all concurrent accesses and provides a [`locked_table`](https://efficient.github.io/libcuckoo/classlibcuckoo_1_1cuckoohash__map_1_1locked__table.html) view over the underlying data structure
providing iterators and an interface more or less equivalent to that of `std::unordered_map`.
This functionality is meant to support scenarios such as concurrent population followed by
non-concurrent usage, for instance.

