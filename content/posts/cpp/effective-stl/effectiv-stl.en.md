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

**Item 1: Choose you containers with care.**

- Standard STL sequence containers: `vector`, `string`, `deque`, and `list`.

- Standard STL associative containers: `set`, `multiset`, `map`, and `multimap`.

- Nonstandard sequence containers: `slist` and `rope`.

- Nonstandard associative containers: `hash_set`, `hash_multiset`, `hash_map`, and `hash_multimap`.

- Standard non-STL containers: arrays, `bitset`, `valarray`, `stack`, `queue`, and `priority_queue`.

**Item 2: Beware the illusion of container-independent code.**

- The different containers are different, and they have strengths and weaknesses that vary in significant ways. They're not designed to be interchangeable, and there's little you can d to paper that over.

**Item 3: Make copying cheap and correct for objects in containers.**

- An easy way to make copying efficient, correct, and immune to the slicing problem is to create containers of pointers instead of containers of objects.

- Compared to arrays, STL containers are much more civilized. They create (by copying) only as many objects as you ask for, they do it only when you direct them to, and they use a default constructor only when you say they should do.

**Item 4: Call `empty` instead of checking `size()` against zero.**

- `empty` is typically implemented as an inline function that simply returns whether `size` returns 0.

- `empty` is a constant-time operation for all standard containers, but for some `list` implementations, `size` may take linear time.

```c++
std::list<int> list1{1, 2, 3, 4, 5, 6, 7, 8 ,9, 10};
std::list<int> list2{1, 2, 3, 4, 5, 6, 7, 8 ,9, 10};

// But how many elements were spliced into list1?
list1.splice(list1.end(), std::find(list2.begin(), list2.end(), 5),
             std::find(list2.begin(), list2.end(), 10).base());
```

**Item 5: Prefer range member functions to their single-element counterparts.**

- Whenever you have to completely replace the contests of a container, you should think of assignment.

- Almost all uses of `copy` where the destination range is specified using an insert iterator can be -- should be -- replaced with calls to range member functions.

- Range member functions are easier to write, they express your intent more clearly, and they exhibit higher performance.

```c++
constexpr int numValues = 100;
int data[numValues];
std::vector<int> v;

// Range-based style.
v.insert(v.begin(), data, data + numValues);

// Iterative-based style.
std::vector<int>::iterator insertLoc(v.begin());
for (int i = 0; i < numValues; ++i) {
  insertLoc = v.insert(insertLoc, data[i]);
  ++insertLoc;
}

// Tax 1: inserting numValues elements into v one at a time naturally costs you numValues calls to insert.
// Tax 2: Each time insert is called to add a new value to v, every element above the insertion point must be moved up one position to make room for the new element.
// Tax 3: memory reallocation.
```

**Item 6: Be alert for C++'s most vexing parse.**

- Parentheses around parameter name are ignored, but parentheses standing by themselves indicate the existence of a parameter list; they announce the presence of a parameter that is itself a pointer to a function.

- Pretty much anything that can be parsed as a function declaration will be.

```c++
std::ifstream dataFile("ints.dat");

// The function data takes two parameters:
// The first parameter is named dataFile. Its type is istream_iterator<int>. The parentheses around dataFile are superfluous and are ignored.
// The second parameter has no name. Its type is pointer to function taking nothing and returning an istream_iterator<int>.
std::list<int> data(std::istream_iterator<int>(dataFile), std::istream_iterator<int>());

// Correct way:
std::list<int> data((std::istream_iterator<int>(dataFile)), std::istream_iterator<int>());
```

**Item 7: When using containers of `new`ed pointers, remember to `delete` the pointers before the container is destroyed.**

- A container of pointers will destroy each element it contains when it (the container) is destroyed, but the "destructor" for a pointer is a no-op!

```c++
// Solution 1: for_each(vwp.begin(), vwp.end(), DeleteObject<Widge>())
template <typename T>
struct DeleteObject : public unary_function<const * T, void> {
  void operator()(const T* ptr) const { delete ptr; }
};

// Solution 2: for_each(vwp.begin(), vwp.end(), DeleteObject())
struct DeleteObject {
  template <typename T>
  void operator()(const T* ptr) const {
    delete ptr;
  }
};
```

**Item 8: Never create containers of `auto_ptr`s.**

- When you copy an `auto_ptr`, ownership of the object pointed to by the `auto_ptr` is transferred to the copying `auto_ptr`, and the copied `auto_ptr` is set to `NULL`.

```c++
std::vector<std::vector<Widget>> widgets;

bool widgetAPCompare(const std::auto_ptr<Widget>& lhs, const std::auto_ptr<Widget>& rhs) {
  return *lhs < *rhs;
}

// By the time the call to sort returns, the contents of the vector will be changed, and at least one Widget will have been deleted.
std::sort(widgets.begin(), widgets.end(), widgetAPCompare);
```

**Item 9: Choose carefully among erasing options.**

- To eliminate all objects in a container that have a particular value:

  - If the container is a `vector`, `string`, or `deque`, use the `erase-remove` idiom.
  - If the container is a `list`, use `list::remove`.
  - If the container is a standard associative container, use its `erase` member function.

- To eliminate all objects in a container that satisfy a particular predicate:

  - If the container is a `vector`, `string`, or `deque`, use the `erase-remove_if` idiom.
  - If the container is a `list`, use `list::remove_if`.
  - If the container is a standard associative container, use `remove_copy_if` and `swap`, or write a loop to walk the container elements, being sure to postincrement your iterator when you pass it to `erase`.

- To do something inside the loop (in addition to erasing objects):

  - If the container is a standard sequence container, write a loop to walk the container elements, being sure to update your iterator with `erase`'s return value each time you call it.
  - If the container is a standard associative container, write a loop to walk the container elements, being sure to postincrement your iterator when you pass it to `erase`.

**Item 10: Be aware of allocator conventions and restrictions.**

- All allocator objects of the same type are equivalent and always compare equal.

- If you ever want to write a custom allocator:

  - Make your allocator a template, with the template parameter `T` representing the type of objects for which you are allocating memory.
  - Provide the typedefs `pointer` and `reference`, but always have `pointer` be `T*` and `reference` be `T&`.
  - Never give your allocators per-object state. In general, allocators should have no nonstatic data members.
  - Remember that an allocator's `allocate` member functions are passed the number of objects for which memory is required, not the number of bytes needed. Also remember that these functions return `T*` pointers (via the `pointer` typedef), even though no `T` objects have yet been constructed.
  - Be sure to provide the nested `rebind` template on which standard containers depend.

```c++
template <typename T>
class allocator {
 public:
  template <typename U>
  struct rebind {
    typedef allocator<U> other;
  };
  // ...
};

// What we need is the memory for a ListNode that contains a T.
template <typename T, typename Allocator = allocator<T>>
class list {
 private:
  Allocator alloc;   // allocator for Ts is Allocator
  struct ListNode {  // allocator for ListNodes is Allocator::rebind<ListNode>::other
    T data;
    ListNode* prev;
    ListNode* next;
  };
  // ...
};
```

**Item 11: Understand the legitimate uses of custom allocators.**

- Employ custom allocators to control general memory management strategies, clustering relationships, and use of shared memory and other special heaps.

```c++
// Legitimate use 1: shared-memory allocator
void* mallocShared(size_t bytesNeeded);
void freeShared(void* ptr);

template <typename T>
class SharedMemoryAllocator {
 public:
  pointer allocator(size_type numObjects, const void* localityHint = 0) {
    return static_cast<pointer>(mallocShared(numObjects * sizeof(T)));
  }

  void deallocate(pointer ptrToMemory, size_type numObjects) { 
      freeShared(ptrToMemory);
  }
};

typedef std::vector<double, SharedMemoryAllocator<double>> SharedDoubleVec;
SharedDoubleVec v;

// Legitimate use 2: heap specification
class Heap1 {
  static void* alloc(size_t numBytes, const void* memoryBlockToBeNear);
  static void dealloc(void* ptr);
};

class Heap2 {
  static void* alloc(size_t numBytes, const void* memoryBlockToBeNear);
  static void dealloc(void* ptr);
};

template <typename T, typename Heap>
class SpecificHeapAllocator {
 public:
  pointer allocator(size_type numObjects, const void* localityHint = 0) {
    return static_cast<pointer>(Heap::alloc(numObjects * sizeof(T)));
  }

  void deallocate(pointer ptrToMemory, size_type numObjects) { 
      Heap::dealloc(ptrToMemory);
  }
};

std::vector<int, SpecificHeapAllocator<int, Heap1>> v;
std::set<int, SpecificHeapAllocator<int, Heap1>> s;

std::list<Widget, SpecificHeapAllocator<Widget, Heap2>> L;
std::map<int, std::string, std::less<int>, SpecificHeapAllocator<std::pair<const int, std::string>, Heap2>> m;
```

**Item 12: Have realistic expectations about the thread safety of STL containers.**

- For multithreading in STL containers, what you can hope for, not what you can expect:

  - Multiple readers are safe.
  - Multiple writers to different containers are safe.

- C++ guarantees that local objects are destroyed if an exception is thrown, so Lock will release its mutex even if an exception is thrown while we're using the `Lock` object.

```c++
template <typename Container>
class Lock {
 public:
  Lock(Container& container) : c(container) { getMutexFor(c); }
  ~Lock() { releaseMutexFor(c); }

 private:
  const Container& c;
};
```

## CH2: `vector` and `string`

Item 13: Prefer `vector` and `string` to dynamically allocated arrays.

- Any time you find yourself getting ready to dynamically allocate an array (i.e., plotting to write “`new T[...]`”), you should consider using a `vector` or a `string` instead.

Item 14: Use `reserve` to avoid unnecessary reallocations.

- There are two common ways to use `reserve` to avoid unneeded reallocations:

  - The first is applicable when you know exactly or approximately how many elements will ultimately end up in your container.
  - The second way is to reserve the maximum space you could ever need, then, once you've added all your data, trim off any excess capacity.

Item 15: Be aware of variations in `string` implementations.

- string values may or may not be reference counted. By default, many implementations do use reference counting, but they usually offer a way to turn it off, often via a preprocessor macro.

- `string` objects may range in size from one to at least seven times the size of `char*` pointers.

- Creation of a new string value may require zero, one, or two dynamic allocations.

- `string` objects may or may not share information on the string's size and capacity.

- `string`s may or may not support per-object allocators.

- Different implementations have different policies regarding minimum allocations for character buffers.

Item 16: Know how to pass `vector` and `string` data to legacy APIs.

- The return type of `begin` is an iterator, not a pointer, and you should never use `begin` when you need to get a pointer to the data in a `vector`.

- If you pass a `vector` to  C API that modifies its elements, that's typically okay, but the called routine must not attempt to change the number of elements in the vector.

```c++
// In both cases, the pointers being passed are pointers to const. The vector or string data are
// being passed to an API that will read it, not modify it.
void doSomething(const int* pInts, std::size_t numInts);
void doSomething(const char* pString);

// Initialize a vector with elements from a C API.
constexpr int maxNumDoubles = 100;
std::size_t fillArray(double* pArray, std::size_t arraySize);
std::vector<double> vd(maxNumDoubles);
vd.resize(fillArray(&vd[0], vd.size()));

// Initialize a string with elements from a C API.
constexpr int maxNumChars = 100;
std::size_t fillString(char* pArray, std::size_t arraySize);
std::vector<char> vc(maxNumChars);
std::size_t charsWritten = fillString(&vc[0], vc.size());
std::string s(vc.begin(), vc.begin() + charsWritten);
```

Item 17: Use "the `swap` trick" to trim excess capacity.

- There's no guarantee that this technique will truly eliminate excess capcaity.

- A variant of the `swap` trick can be used both to clear a container and to reduce its capacity to the minimum your implementation offers.

```c++
class Contestant;
std::vector<Contestant> contestants;

std::vector<Contestant>(contestants).swap(contestants);  // trim
std::vector<Contestant>().swap(contestants);             // clear and minimize its capacity
```

Item 18: Avoid using `vector<bool>`.

- If `c` is container of objects of type `T` and `c` supports `operator[]`, the statement `T* p = &c[0];` must compile.

- `vector<bool>` is a pseudo-container that contains not actual `bool`s, but a packed representation of `bool`s that is designed to save space.

- `vector<bool>` doesn't satisfy the requirements of an STL container; you're best off not using it; and `deque<bool>` and `bitset` are alternative data structures that will almost certainly satisfy your need for the capabilities promised by `vector<bool>`.

```c++
template <typename Allocator>
std::vector<bool, Allocator> {
 public:
  class reference {};

  reference operator[](std::size_type n);

  // ...
};

std::vector<bool> v;
bool *pb = &v[0];  // the type of &v[0] is std::vector<bool>::reference*, not bool*
```

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
