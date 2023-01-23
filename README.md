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
There are non-standard overloads of `insert` accepting a key rather than a full value —so,
`insert(acc, k)` behaves as (non-existent) `try_emplace(acc, k)`.
`operator[]` is not provided. `erase([const_]iterator)` is logically replaced by
`erase([const_]accessor)`.

`rehash` is thread-safe. Additional functionality is provided _without
thread safety guarantees_:
* iterators,
* assignment,
* `clear` and `swap`.

Parallel iteration is served through [_container ranges_](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/named_requirements/containers/container_range.html),
which behave like C++17 ranges but provide additional splitting functionalities to
distribute the range load traversal across threads. Parallel iteration is
not thread-safe, that is, when parallel-iterating a container no other
operations can be performed on the container.

### libcuckoo

[`libcuckoo::cuckoohash_map`](https://efficient.github.io/libcuckoo/classlibcuckoo_1_1cuckoohash__map.html)
does not have iterators. The thread-safe lookup/modify interface is as follows:
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

Assignment is not thread-safe. `rehash`, `reserve`, `clear` and `swap`, on the other hand,
can be called in a concurrent scenario.

`lock_table` blocks all concurrent accesses and returns a [`locked_table`](https://efficient.github.io/libcuckoo/classlibcuckoo_1_1cuckoohash__map_1_1locked__table.html) view over the underlying data structure
providing iterators and an interface more or less equivalent to that of `std::unordered_map`.
This functionality is meant to support scenarios such as concurrent population followed by
non-concurrent usage, for instance.

### `std::concurrent_unordered_map` proposal

[P0652R3](https://wg21.link/p0652r3) proposes an interface for an (as of yet not accepted)
`std::concurrent_unordered_map` container. `std::concurrent_unordered_map` does not have
iterators. More controversially, it also omits `size`, `count` and `empty` on the grounds
that, in a multithreaded scenario, the values returned can not be trusted.
The thread-safe lookup/modify interface is split into _visitation member
functions_ (accepting a user-provided functor for access to the elements) and non-visitation
member functions. The visitation interface is as follows:
* `void visit(const key_type&, Visitor&)` (const and non-const)
* `void visit_all(Visitor&)`  (const and non-const)
* `bool visit_or_emplace(K&&, Visitor&, Args&&...)` (non-const only)

Visitors are passed const/non-const references to the mapped value type depending on
whether the const or non-const version of the member function is used, respectively
—internally, this implies that either shared or exclusive locking is used.
`visit_all` also passes a `const key_type&` to the visitor.
`visit_or_emplace` only calls the visitor if an equivalent element already existed,
and returns `true` iff the element has been newly created. 

The non-visitation interface is:
* `optional<mapped_type> find(const key_type&)`
* `mapped_type find(const key_type&, Args&&...)`
* `bool emplace(K&& key, Args&&...)`
* `bool insert_or_assign(K&&, Args&&...)`
* `size_type update(const key_type&, Args&&...)`
* `size_type erase(const key_type&)`

`mapped_type find(const key_type& key, Args&&... args)` returns a copy of the mapped value type
of the element with equivalent to `key`, if it exists, and else it returns a `mapped_type`
constructed from `args...`: the intended usage of this member function escapes
the author of these notes. `emplace` and `insert_or_assign` behave as
their homonyms in `std::unordered_map` except that they don't return an iterator.
`update` replaces the mapped value type of the looked-up element and does
nothing otherwise (that is, it does not create an element like is the case
with `insert_or_assign`); as it happens, `update` can be implemented as a
wrapper over non-const `void visit(const key_type&, Visitor&)`.
`operator[]` is not provided.

All operations of `std::concurrent_unordered_map` are thread-safe,
including assignment, `merge`, `swap` and `clear`. No rehashing facilities
are provided.

`make_unordered_map_view(bool)` returns an `unordered_map_view` over the underlying
data structure (with or without blocking of other concurrent access to the
parent `concurrent_unordered_map`, as specified by the `bool` parameter).
`unordered_map_view` interface mimics that of `std::unordered_map`, without the
usual construction and assignment operations, and with some
deviations on _"iterator invalidation requirements, load_factor functions, `size()`
complexity requirements, buckets and node operations"_ —the proposal, which is
clearly incomplete, gets rather vague at this point, as these deviations are not
explained, and the synopsis offered includes funcionality typically associated
to closed-addressing implementations, like a full-fledged bucket interface
with local iterators.

### Comparison table

|  |oneTBB|libcuckoo|`std::concurrent_unordered_map`<br/> proposal|
|:--|:-:|:-:|:-:|
|Iterators|unsafe|no|no<br/><sup>(whole container traversal with `visit_all`)</sup>|
|Assignment|unsafe|unsafe|safe|
|Rehash|safe|safe|no|
|`clear`, `swap`|unsafe|safe|safe|
|`size`, `count`, `empty`|safe|safe|no|
|`operator[]`|no|no|no|
|Lookup/modify interface|Adapted classical interface plus<br>accessor-based overloads|Adapted classical interface plus<br>functor-based `find_fn`, `update_fn`,<br/>`uprase_fn`, `upsert`|Adapted classical interface plus<br>`visit`, `visit_or_emplace`,<br/>`update`| 
|Thread-unsafe view|no|yes<br/><sup>(locks parent container)</sup>|yes<br/><sup>(parent locking specified by user)</sup>|
