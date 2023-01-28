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

`find(key, args...)` returns a copy of the mapped value type
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

`make_unordered_map_view(lock)` returns an `unordered_map_view` over the underlying
data structure (the `bool` parameter `lock` specifies whether concurrent accesses to
the parent `concurrent_unordered_map` are blocked or not).
`unordered_map_view` interface mimics that of `std::unordered_map`, without the
usual construction and assignment operations, and with some
deviations on _"iterator invalidation requirements, load_factor functions, `size()`
complexity requirements, buckets and node operations"_ —the proposal, which is
clearly incomplete, gets rather vague at this point, as these deviations are not
explained, and the synopsis offered includes funcionality typically associated
to closed-addressing implementations, like a full-fledged bucket interface
with local iterators.

### gtl 

[gtl parallel hash containers](https://github.com/greg7mdp/gtl/blob/main/docs/phmap.md)
include not only concurrent maps, but also concurrent sets, and node-based variations of those:
for brevity, the rest of the discussion focuses on `gtl::parallel_flat_hash_map`.

`gtl::parallel_flat_hash_map` provides a classical interface mimicking that of `absl::flat_hash_map`,
and most member functions properly lock the underlying data structure on execution,
but the iterators _are not thread-safe_ (they are not locked on the pointed to
element): so, iterator-based functionalities and member functions returning
an element reference (`operator[]`) are esentially thread-unsafe. Additionally,
the following thread-safe, function-based operations are provided:
* `bool if_contains(const K&, F&&)`
* `bool modify_if(const K&, F&&)`
* `bool erase_if(const K&, F&&)`
* `bool try_emplace_l(K&&, F&&, Args&&...)`
* `bool lazy_emplace_l(const K&, FExists&&, FEmplace&&)`
* `void for_each(F&&)` (const) 
* `void for_each_m(F&&)` (non-const)
* `void with_submap(size_t, F&&)` (const)
* `void with_submap_m(size_t, F&&)` (non-const)

`if_contains(key, f)` and `modify_if(key, f)` provide read and write access, respectively,
to the element equivalent to `key`. `erase_if(key, f)` erases the element `x`
equivalent to `key` iff `f(x)` returns true.
`try_emplace_l(k, f, args...)` inserts an element constructed with
(`k`, `args...`) and returns `true`, or invokes `f(x)` and returns `false`
if there's a preexistent equivalent element `x`.
`lazy_emplace_l(key, fExists, fEmplace)` looks for
an element equivalent to `key`: if it exists, invokes `fExists` on it, otherwise,
a so-called _constructor object_ is passed to `fEmplace` to allow for
element creation:

```cpp
map.lazy_emplace_l(
  5,
  [](tbb_map_type::value_type& x){ x.second = 6; },     // update
  [](tbb_map_type::constructor& ctor){ ctor(5, 13); }); // or create
```

(Incidentally, the implementation does not check that the element created is
actually equivalent to `key`, which can lead to a violation of the container
invariants). `for_each` and `for_each_m` allow for read and write traversal
of the container, respectively, whereas `with_submap` and `with_submap_m` perform
_submap_ traversal (`gtl::parallel_flat_hash_map` implementation is based on
sharding).

### Comparison table

||oneTBB|libcuckoo|P0652R3 proposal|gtl|
|:--|:-:|:-:|:-:|:-:|
|Containers provided|map|map|map|map, set and node-based variations|
|Iterators|unsafe|no|no<br/><sup>(safe traversal with `visit_all`)</sup>|unsafe<br/><sup>(safe traversal with `for_all[_m]`)</sup>|
|Assignment|unsafe|unsafe|safe|unsafe|
|Rehash|safe|safe|no|safe|
|`clear`, `swap`|unsafe|safe|safe|safe|
|`size`, `count`, `empty`|safe|safe|no|safe|
|`operator[]`|no|no|no|unsafe|
|Lookup/modify interface|adapted classical interface plus accessor-based overloads|adapted classical interface plus functor-based `find_fn`, `update_fn`, `uprase_fn`, `upsert`|adapted classical interface plus `visit`, `visit_or_emplace`, `update`|unsafe classical interface plus functor-based `if_contains`, `modify_if`, `erase_if`, `try_emplace_l`, `lazy_emplace_l`|
|Parallel iteration|with splittable ranges (unsafe)|no explicit support|no explicit support|with `with_submap[_m]` (safe)|
|Thread-unsafe view|no|yes<br/><sup>(locks parent container)</sup>|yes<br/><sup>(parent locking specified by user)</sup>|no|

## Design guidelines

* The entire map interface must be thread-safe, including assignment and destruction.
In the case of assignment, the implementation must be protected against potential
deadlock scenarios such as the following:
```cpp
// Thread 1
map1=map2;

// Thread 2
map2=map1;
```

* Don't provide iterators or accessors, either blocking or not: if not-blocking, they're
unsafe, and if blocking they increase contention when not properly used,
and can very easily lead to deadlocks:
```cpp
// Thread 1
map_type::iterator it1=map.find(x1), it2=map.find(x2);

// Thread 2
map_type::iterator it2=map.find(x2), it1=map.find(x1);
```
* Don't provide `operator[]` (thread-unsafe). This is syntactic sugar and can be
replaced by safe alternatives.
* Provide `size`, `empty`, `count`. Even though P0652R3 deems these member functions
useless and dangerous, they don't behave differently to any other
lookup/modify method in that the status of the container may change as
soon as the operation completes. Moreover, this information is useful
and reliable in the second stage of common population+access scenarios.
* Access to elements shall be in read or write mode based on whether the
member function used is const or not, respectively.
* Provides some means of doing parallel iteration.

## Open questions

* In addition to `concurrent_hash_map`, do we want any of `concurrent_hash_set`
and `concurrent_hash_node_[map|set]`?
* A variation of `concurrent_hash_node_map` can internally store
`std::shared_ptr<value_type>`s and provide accesor-like, non-blocking, thread-safe
access to the elements (without the guarantee that elements remain in the map during
the accessor lifetime). Is this a valuable data structure?
* Do we want to provide a (blocking or not) thread-unsafe view over the map, in the
same way as libcukcoo and P0652R3 do? There's the additional possibility of
providing O(1) move assignment from `concurrent_hash_map` to `unordered_flat_map`
(move assignment in the opposite direction would be O(n) but fairly cheap).

## A systematic approach to designing our lookup/modify interface

Iterators allow for composition of several operations on the same element
—for instance, find an element and erase it or not based on some check on its
mapped value. In the absence of iterators, composite operations are not possible
(elements can't be referenced once a basic map operation has completed), so
we need to extend the map interface to cover most (hopefully all) composite
operations natively. The following is an attempt at designing such interface.

There are only four basic operations or "primitives" one can perform on a value/element:
**FIND**, **ACCESS**, **INSERT** and **ERASE** (read and write access are generically
covered by **ACCESS**, and **INSERT** comprises both insertion and emplacement).
We can derive an exhaustive diagram of how these operations can be meaningfully
composed over the same element, which naturally yields our extended map interface:

![diagram](lookup_modify.png)

Some observations for discussion:

* `find(k)` is entirely equivalent to `contains(k)`, so we may consider removing it.
* `find(k, f)` could also be named `visit` à la P0652R3.
* Similarly, all `*_modify` operations could be named `*_visit`. `access_and_erase`
can be named `moodify_and_erase` or `visit_and_erase`.
* `*_modify(..., f, ...)` operations can be trivially emulated with
`*_erase_if(..., [&](auto& x){ f(x); return false; }, ...)`, but it seems reasonable
to keep `*_modify` as their intent is quite different.
* `*_or_erase[_if]` operations serve a rather exotic scenario (erase an
element if it caused a collision on insertion) and could be emulated by an insertion
operation followed by `erase_if` on failure, although less efficiently
and in a non-atomic manner (which is probably unobservable anyway). We should decide
whether we keep them for completeness or if we drop them. 

## On parallel iteration

TBW [TBB vs. std, iterators not being forward, blocking vs. non-blocking]
