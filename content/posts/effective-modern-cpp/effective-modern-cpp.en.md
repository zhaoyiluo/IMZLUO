---
title: "Effective Modern Cpp"
subtitle: "42 Specific Ways to Improve Your Use of C++11 and C++14"
date: 2021-08-22T22:44:11+08:00
lastmod: 2021-08-22T22:44:11+08:00
draft: false
authors: ["zhaoyiluo"]
description: ""

tags: []
categories: ["C++"]
series: []

featuredImage: ""
featuredImagePreview: ""
---

## CH1: Deducing Types

**Item 1: Understand template type deduction.**

**Item 2: Understand `auto` type deduction.**

**Item 3: Understand `decltype`.**

**Item 4: Know how to view deduced types.**

## CH2: `auto`

**Item 5: Prefer `auto` to explicitly type declarations.**

**Item 6: Use the explicitly typed initializer idiom when `auto` deduces undesired types.**

## CH3: Moving to Modern C++

**Item 7: Distinguish between `()` and `{}` when creating objects.**

**Item 8: Prefer `nullptr` to `0` and `NULL`.**

**Item 9: Prefer alias declarations to `typedef`s.**

**Item 10: Prefer scoped `enum`s to unscoped `enum`s.**

**Item 11: Prefer deleted functions to private undefined ones.**

**Item 12: Declare overriding functions `override`.**

**Item 13: Prefer `const_iterator`s to `iterator`s.**

**Item 14: Declare functions `noexcept` if they won't emit exceptions.**

**Item 15: Use `constexpr` whenever possible.**

**Item 16: Make `const` member functions thread safe.**

**Item 17: Understand special member function generation.**

## CH4: Smart Pointers

**Item 18: Use `std::unique_ptr` for exclusive-ownership resource management.**

**Item 19: Use `std::shared_ptr` for shared-ownership resource management.**

**Item 20: Use `std::weak_ptr` for `std::shared_ptr`-like pointers that can dangle.**

**Item 21: Prefer `std::make_unique` and `std::make_shared` to direct use of `new`.**

**Item 22: When using the Pimpl Idiom, define special member functions in the implementation file.**

## CH5: Rvalue References, Move semantics, and Perfect Fowarding

**Item 23: Understand `std::move` and `std::forward`.**

**Item 24: Distinguish universal references from rvalue references.**

**Item 25: Use `std::move` on rvalue references, `std::forward` on universal references.**

**Item 26: Avoid overloading on universal references.**

**Item 27: Familiarize yourself with alternatives to overloading on universal references.**

**Item 28: Understand reference collapsing.**

**Item 29: Assume that move operations are not present, not cheap, and not used.**

**Item 30: Familiarize yourself with perfect forwarding failure cases.**

## CH6: Lambda Expressions

**Item 31: Avoid default capture modes.**

**Item 32: Use init capture to move objects into closures.**

**Item 33: Use `decltype` on `auto&&` parameters to `std::forward` them.**

**Item 34: Prefer lambdas to `std::bind`.**

## CH7: The Concurrency API

**Item 35: Prefer task-based programming to thread-based.**

**Item 36: Specify `std::launch::async` if asynchronicity is essential.**

**Item 37: Make `std::threads` unjoinable on all paths.**

**Item 38: Be aware of varying thread handle destructor behavior.**

**Item 39: Consider `void` futures for one-shot event communication.**

**Item 40: Use `std::atomic` for concurrency, `volatile` for special memory.**

## CH8: Tweaks

**Item 41: Consider pass by value for copyable parameters that are cheap to move and always copied.**

**Item 42: Consider emplacement instead of insertion.**