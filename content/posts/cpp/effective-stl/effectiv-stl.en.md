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

**Item 13: Prefer `vector` and `string` to dynamically allocated arrays.**

- Any time you find yourself getting ready to dynamically allocate an array (i.e., plotting to write “`new T[...]`”), you should consider using a `vector` or a `string` instead.

**Item 14: Use `reserve` to avoid unnecessary reallocations.**

- There are two common ways to use `reserve` to avoid unneeded reallocations:

  - The first is applicable when you know exactly or approximately how many elements will ultimately end up in your container.
  - The second way is to reserve the maximum space you could ever need, then, once you've added all your data, trim off any excess capacity.

**Item 15: Be aware of variations in `string` implementations.**

- string values may or may not be reference counted. By default, many implementations do use reference counting, but they usually offer a way to turn it off, often via a preprocessor macro.

- `string` objects may range in size from one to at least seven times the size of `char*` pointers.

- Creation of a new string value may require zero, one, or two dynamic allocations.

- `string` objects may or may not share information on the string's size and capacity.

- `string`s may or may not support per-object allocators.

- Different implementations have different policies regarding minimum allocations for character buffers.

**Item 16: Know how to pass `vector` and `string` data to legacy APIs.**

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

**Item 17: Use "the `swap` trick" to trim excess capacity.**

- There's no guarantee that this technique will truly eliminate excess capcaity.

- A variant of the `swap` trick can be used both to clear a container and to reduce its capacity to the minimum your implementation offers.

```c++
class Contestant;
std::vector<Contestant> contestants;

std::vector<Contestant>(contestants).swap(contestants);  // trim
std::vector<Contestant>().swap(contestants);             // clear and minimize its capacity
```

**Item 18: Avoid using `vector<bool>`.**

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

**Item 19: Understand the difference between equality and equivalence.**

- `find`'s definition of "the same" is equality, which is based on `operator==`. `set::insert`'s definition of "the same" is equivalence, which is usually based on `operator<`.

- Equivalence is based on the relative ordering of object values in a sorted range and every standard associative container makes its sorting predicate available through its `key_comp` member function.

- Once you leave the realm of sorted associative containers, the situation changes, and the issue of equality versus equivalence can be -- and has been -- revisited.

```c++
bool ciStringCompare(const std::string& lhs,
                     const std::string& rhs);  // case-insensitive string comparisons

struct CIStringCompare : public std::binary_function<std::string, std::string, bool> {
  bool operator()(const std::string& lhs, const std::string& rhs) const {
    return ciStringCompare(lhs, rhs);
  }
};

std::set<std::string, CIStringCompare> ciss;
ciss.insert("Persephone");  // a new element is added to the set
ciss.insert("persephone");  // no new element is added to the set

if (ciss.find("persephone") != ciss.end()) {
  // this test will succeed
}

if (std::find(ciss.begin(), ciss.end(), "persephone") != ciss.end()) {
  // this test will fail
}
```

**Item 20: Specify comparison types for associative containers of pointers.**

- Anytime you create a standard associative container of pointers, you must bear in mind that the container will be sorted by the values of the pointers.

- You'll almost always want to create your own functor class to serve as a comparison type.

```c++
struct StringPtrLess : public std::binary_function<const std::string*, const std::string*, bool> {
  bool operator()(const std::string* ps1, const std::string* ps2) const { return *ps1 < *ps2; }
};

typedef std::set<std::string*, StringPtrLess>
    StringPtrSet;  // each of the set template's parameters is a type, not a function
StringPtrLess ssp;

// Solution 1
for (StringPtrLess::const_iterator i = ssp.begin(); i != ssp.end(); ++i) {
  std::cout << **i << std::endl;
}

// Solution 2
void print(const std::string* ps) { std::cout << *ps << std::endl; }
std::for_each(ssp.begin(), ssp.end(), print);

// Solution 3
struct Deference {
  template <typename T>
  const T& operator()(const T* ptr) const {
    return *ptr;
  }
};
std::transform(ssp.begin(), ssp.end(), std::ostream_iterator<std::string>(std::cout, "\n"),
               Deference());
```

**Item 21: Always have comparison functions return `false` for equal values.**

- Equal values never precede one another, so comparison functions should always return `false` for equal values.

- Comparison functions used to sort associative containers must define a "strict weak ordering" over the objects they compare. The above is one of the requirements.

```c++
std::set<int, std::less_equal<int>> s;
s.insert(10);  // 10A
s.insert(10);  // 10B

// Check expression: !(10A <= 10B) && !(10B <= 10A), which returns false.
// It means 10A and 10B are not equivalent, hence not the same, and it thus goes about inserting 10B into the container alongside 10A.
```

**Item 22: Avoid in-place key modification in `set` and `multiset`.**

- For objects of type `set<T>` or `multiset<T>`, the type of the elements stored in the container is simply `T`, not `const T`. If you do change an element in a `set` or `multiset`, you must be sure not to change a key part -- a part of the element that affects the sortedness of the container.

- An implementation could have `operator*` for a `set<T>::iterator` return a `const T&`. That is, it could have the result of dereferencing a `set` iterator be a reference-to-`const` element of the set. Thus, code that attempts to modify elements in a `set` or `multiset` isn't portable.

- With `set` and `multiset`, if you perform any in-place modifications of container elements, you are responsible for making sure that the container remains sorted.

- A `map<K, V>` or a `multimap<K, V>` contains elements of type `pair<const K, V>`. That `const` means that the first component of the pair is defined to be `const`, and that means that attempts to modify it (even  after casting away its `const`ness) are undefined.

```c++
class Employee;
struct IDNumberLess : public std::binary_function<Employee, Employee, bool> {
  bool operator()(const Employee& lhs, const Employee& rhs) {
    return lhs.idNumber() < rhs.idNumber();
  }
};

typedef std::set<Employee, IDNumberLess> EmpIDSet;
EmpIDSet se;

Employee selectedID;
EmpIDSet::iterator i = se.find(selectedID);

if (i != se.end()) {
  i->setTitle("Corporate Deity");  // some STL implementations will reject this line because *i is const
  const_cast<Employee&>(*i).setTitle("Corporate Deity");  // treat the result of the cast as a reference to a (non-const) Employee
}
```

**Item 23: Consider replacing associative containers with sorted `vector`s.**

- Sorting data in a sorted `vector` is likely to consume less memory than sorting the same data in a standard associative container, and searching a sorted `vector` vis binary search is likely to be faster than searching a standard associative container when page faults are taken into account.

- To emulate a `map` or `multimap` using a `vector`, you must omit the `const`, because when you sort the vector, the values of its elements will get moved around via assignment, and that means that both components of the par must be assignable.

```c++
typedef std::pair<std::string, int> Data;

class DataCompare {
 public:
  bool operator()(const Data& lhs, const Data& rhs) const {
      return keyLess(lhs.first, rhs.first);
  }

  bool operator()(const Data& lhs, const Data::first_type& k) const {
    return keyLess(lhs.first, k);
  }

  bool operator()(const Data::first_type& k, const Data& rhs) const {
    return keyLess(k, rhs.first);
  }

 private:
  bool keyLess(const Data::first_type& k1, const Data::first_type& k2) const { return k1 < k2; }
};
```

**Item 24: Choose carefully between `map::operator[]` and `map::insert` when efficiency is important.**

- For the expression `m[k] = v;`, `map::operator[]` returns a reference to the value object associated with `k`. `v` is then assigned to the object to which the reference (the one returned from `operator[]`) refers.

  - If `k` isn't yet in the map, it creates one from scratch by using the value type's default constructor. `operator[]` then returns a reference to this newly-created object.

- `insert` is preferable to `operator[]` when adding an element to a map, and both efficiency and aesthetics dictate that `operator[]` is preferable when updating the value of an element that's already in the map.

```c++
// If we had used MapType::mapped_type as the type of efficientAddOrUpdate's third parameter, we'd have converted double to a Widget at the point of call, and we'd thus have paid for a Widget construction we didn't need.
std::map<int, Widget> m;

class Widget {
 public:
  // ...
  Widget& operator=(double weight);
  // ...
};

// KeyArgType and ValueArgType need not be the types sorted in the map. They need only be convertible to the types stored in the map.
template <typename MapType, typename KeyArgType, typename ValueArgType>
typename MapType::iterator efficientAddOrUpdate(MapType& m, const KeyArgType& k,
                                                const ValueArgType& v) {
  typename MapType::iterator lb = m.lower_bound(k);
  if (lb != m.end() && !(m.key_comp()(k, lb->first))) {
    lb->second = v;
    return lb;
  } else {
    typedef typename MapType::value_type MVT;
    return m.insert(MVT(k, v));
  }
}
```

**Item 25: Familiarize yourself with the nonstandard hashed containers.**

- SGI employs a conventional open hashing scheme composed of an array (the buckets) of pointers to singly linked lists of elements.

- Dinkumware also employs open hashing, but its design is based on a novel data structure consisting of an array of iterators (essentially the buckets) into a doubly linked list of elements, where adjacent pairs of iterators identify the extent of the elements in each bucket.

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
