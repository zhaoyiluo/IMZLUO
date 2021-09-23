---
title: "[NOTE] Effective STL"
subtitle: "50 Specific Ways to Improve Your Use of the Standard Template Library"
date: 2021-09-23T21:52:31+08:00
lastmod: 2021-09-23T21:52:31+08:00
draft: false
authors: ["zhaoyiluo"]
description: ""

tags: []
categories: ["C++"]
series: []

featuredImage: ""
featuredImagePreview: ""
---

## CH1: Containers

Item 1: Choose you containers with care.

Item 2: Beware the illusion of container-independent code.

Item 3: Make copying cheap and correct for objects in containers.

Item 4: Call `empty` instead of checking `size()` against zero.

Item 5: Prefer range member functions to their single-element counterparts.

Item 6: Be alert for C++'s most vexing parse.

Item 7: When using containers of `new`ed pointers, remember to `delete` the pointers before the container is destroyed.

Item 8: Never create containers of `auto_ptr`s.

Item 9: Choose carefully among erasing options.

Item 10: Be aware of allocator conventions and restrictions.

Item 11: Understand the legitimate uses of custom allocators.

Item 12: Have realistic expectations about the thread safety of STL containers.

## CH2: `vector` and `string`

Item 13: Prefer `vector` and `string` to dynamically allocated arrays.

Item 14: Use `reserve` to avoid unnecessary reallocations.

Item 15: Be aware of variations in `string` implementations.

Item 16: Know how to pass `vector` and `string` data to legacy APIs.

Item 17: Use "the `swap` trick" to trim excess capacity.

Item 18: Avoid using `vector<bool>`.

## CH3: Associative Containers

Item 19: Understand the difference between equality and equivalence.

Item 20: Specify comparison types for associative containers of pointers.

Item 21: Always have comparison functions return `false` for equal values.

Item 22: Avoid in-place key modification in `set` and `multiset`.

Item 23: Consider replacing associative containers with sorted `vector`s.

Item 24: Choose carefully between `map::operator[]` and `map::insert` when efficiency is important.

Item 25: Familiarize yourself with the nonstandard hashed containers.

## CH4: Iterators

Item 26: Prefer `iterator` to `const_iterator`, `reverse_iterator`, and `const_reverse_iterator`.

Item 27: Use `distance` and `advance` to convert a container's `const_iterator`s to `iterator`s.

Item 28: Understand how to use a `reverse_iterator`'s base `iterator`.

Item 29: Consider `istreambuf_iterator`s for character-by-character input.

## CH5: Algorithms

Item 30: Make sure destination ranges are big enough.

Item 31: Know your sorting options.

Item 32: Follow `remove`-like algorithms by `erase` if you really want to remove something.

Item 33: Be wary of `remove`-like algorithms on containers of pointers.

Item 34: Note which algorithms expect sorted ranges.

Item 35: Implement simple case-insensitive string comparisons via `mismatch` or `lexicographical_compare`.

Item 36: Understand the proper implementation of `copy_if`.

Item 37: Use `accumulate` or `for_each` to summarize ranges.

## CH6: Functions, Functor Classes, Functions, etc.

Item 38: Design functor classes for pass-by-value.

Item 39: Make predicates pure functions.

Item 40: Make functor classes adaptable.

Item 41: Understand the reasons for `ptr_fun`, `mem_fun`, and `mem_fun_ref`.

Item 42: Make sure `less<T>` means `operator<`.

## CH7: Programming with the STL

Item 43: Prefer algorithm calls to hand-written loops.

Item 44: Prefer member functions to algorithms with the same names.

Item 45: Distinguish among `count`, `find`, `binary_search`, `lower_bound`, `upper_bound`, and `equal_range`.

Item 46: Consider function objects instead of functions as algorithm parameters.

Item 47: Avoid producing write-only code.

Item 48: Always `#include` the proper headers.

Item 49: Learn to decipher STL-related compiler diagnostics.

Item 50: Familiarize yourself with STL-related web sites.
