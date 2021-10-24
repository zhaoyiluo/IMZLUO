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

**Item 26: Prefer `iterator` to `const_iterator`, `reverse_iterator`, and `const_reverse_iterator`.**

- Some versions of `insert` and `erase` require `iterator`s. If you want to call those functions, you're going to have to produce `iterator`s. const and reverse iterators won't do.

- It's not possible to implicitly convert a `const_iterator` to an `iterator`.

- Conversion from a `reverse_iterator` to an `iterator` may require iterator adjustment after the conversion.

**Item 27: Use `distance` and `advance` to convert a container's `const_iterator`s to `iterator`s.**

- For `deque`, `list`, `set`, `multiset`, `map`, `multimap`, and the hashed container types, `iterator` and `const_iterator` are different classes, barely more closely related to one another than `string` and `complex<double>`.

- Because it may take linear time to produce an `iterator` equivalent to a `const_iterator`, and because it can't be done at all useless the container for the `const_iterator` is available when the `const_iterator` is, you may wish to rethink designs that require producing `iterator`s from `const_iterator`s.

```c++
typedef std::deque<int> IntDeque;
typedef IntDeque::iterator Iter;
typedef IntDeque::const_iterator ConstIter;

IntDeque d;
ConstIter ci;

Iter i(d.begin());
std::advance(i, std::distance(i, ci));  // not possible for InputIterator to be two different types at the same time, so the call to distance fails
std::advance(i, std::distance<ConstIter>(i, ci));  // explicitly specify the type parameter to be used by distance
```

**Item 28: Understand how to use a `reverse_iterator`'s base `iterator`.**

- To emulate insertion at a position specified by a `reverse_iterator` `ri`, insert at the position `ri.base()` instead. For purpose of insertion, `ri` and `ri.base()` are equivalent, and `ri.base()` is truly the iterator corresponding to `ri`.

- To emulate erasure at a position specified by a `reverse_iterator` `ri`, erase at the position preceding `ri.base()` instead. For purposes of erasure, `ri` and `ri.base()` are not equivalent, and `ri.base()` is not the `iterator` corresponding to `ri`.

```c++
std::vector<int> v;
v.reserve(5);

for (int i = 0; i < 5; ++i) {
  v.push_back(i);
}

std::vector<int>::reverse_iterator ri = std::find(v.rbegin(), v.rend(), 3);
std::vector<int>::iterator i(ri.base());

v.insert(i, 99);         // for insertion
v.erase((++ri).base());  // for erasure
v.erase(--ri.base());    // won't compile where string and vector iterators are pointers
```

**Item 29: Consider `istreambuf_iterator`s for character-by-character input.**

- The `operator>>` functions on which `istream_iterator`s depend perform formatted input, and that means they must undertake a fair amount of work on your behalf each time you call one.

- For unformatted character-by character input, you should always consider `istreambuf_iterator`s. The same applies to `ostreambuf_iterator`s.

```C++
std::ifstream inputFile("interestingData.txt");
inputFile.unsetf(std::ios::skipws);
std::string fileData((std::istream_iterator<char>(inputFile)), std::istream_iterator<char>());

std::ifstream inputFile("interestingData.txt");
std::string fileData((std::istreambuf_iterator<char>(inputFile)), std::istreambuf_iterator<char>());
```

## CH5: Algorithms

**Item 30: Make sure destination ranges are big enough.**

- Whenever you use an algorithm requiring specification of a destination range, ensure that the the destination range is big enough already or is increased in size as the algorithm runs.

```c++
int transmogrify(int x);

std::vector<int> values;
std::vector<int> results;

// Like every algorithm that uses a destination range, `transform` writes its results by making assignments to the elements in the destination range.
std::transform(values.begin(), values.end(), results.end(), transmogrify);

// Internally, the iterator returned by back_inserter causes push_back to be called, so you may use back_inserter with any container offering push_back.
std::transform(values.begin(), values.end(), std::back_inserter(results), transmogrify);

// Using reserve without also using an insert iterator leads to undefined behavior inside algorithms as well as to corrupted containers.
results.reserve(values.size() + values.size());
std::transform(values.begin(), values.end(), std::back_inserter(results), transmogrify);
```

**Item 31: Know your sorting options.**

- If you need to perform a full sort on a `vector`, `string`, `deque`, or array, you can use `sort` or `stable_sort`.

- If you have a `vector`, `string`, `deque`, or array and you need to put only the top n elements in order, `partial_sort` is available.

- If you have a `vector`, `string`, `deque`, or array and you need to identify the elements at position n or you need to identify the top n elements without putting them in order, `nth_element` is at your beck and call.

- If you need to separate the elements of a standard sequence container or an array into those that do and do not satisfy some criterion, you're probably looking for `partition` or `stable_partition`.

- If your data is in a `list`, you can use `partition` and `stable_partition` directly, and you can use `list::sort` in place of `sort` and `stable_sort`. If you need the effects offered by `partial_sort` or `nth_element`, you'll have to approach the task indirectly, but there are a number of options.

**Item 32: Follow `remove`-like algorithms by `erase` if you really want to remove something.**

- Because the only way to eliminate an element from a container is to invoke a member function on that container, and because `remove` cannot know the container holding the elements on which it is operating, it is not possible for `remove` to eliminate elements from a container.

- Internally, remove walks own the range, overwriting values that are to be "removed" with later values that are to be retained. The overwriting is accomplished by making assignments to the elements holding the values to be overwritten.

- You should follow `remove` by `erase` if you really want to remove something.

- There are two other "`remove`-like" algorithms: `remove_if` and `unique`.

```c++
std::vector<int> v;
v.reserve(10);

for (int i = 1; i <= 10; ++i) {
  v.push_back(i);
}

v[3] = v[5] = v[9] = 99;

std::remove(v.begin(), v.end(), 99);  // v.size() == 10
v.erase(std::remove(v.begin(), v.end(), 99), v.end());  // erase-remove idiom
```

**Item 33: Be wary of `remove`-like algorithms on containers of pointers.**

- To deal with containers of dynamically allocated pointers, be it by reference counting smart pointers, manual deletion and nullification of pointers prior to invoking a `remove`-like algorithm, or some technique of your own invention.

```c++
class Widget {
 public:
  // ...
  bool isCertified() const;
  // ...
};

std::vector<Widget*> v;
v.push_back(new Widget());

// The "removed" pointers have been overwritten by later "unremoved" pointers in the vector. Nothing
// points to the uncertified Widgets, they can never be deleted, and their memory and other
// resources are leaked.
v.erase(std::remove_if(v.begin(), v.end(), std::not1(std::mem_fun(&Widget::isCertified))), v.end());

// Solution 1:
void delAndNullifyUncertified(Widget*& pWidget) {
  if (!pWidget->isCertified()) {
    delete pWidget;
    pWidget = 0;
  }
}

std::for_each(v.begin(), v.end(), delAndNullifyUncertified);
v.erase(std::remove(v.begin(), v.end(), static_cast<Widget*>(0)), v.end());

// Solution 2:
template <typename T>
class RCSP {};  // reference counting smart pointer

typename RCSP<Widget> RCSPW;
std::vector<RCSPW> v;
v.push_back(RCSPW(new Widget));

v.erase(std::remove_if(v.begin(), v.end()), std::not1(std::mem_fun(&Widget::isCertified))), v.end());
```

**Item 34: Note which algorithms expect sorted ranges.**

- The search algorithms, `binary_search`, `lower_bound`, `upper_bound`, and `equal_range`, guarantee logarithmic-time look up only when they are passed random access iterators. If they're given less powerful iterators (such as bidirectional iterators), they still perform only a logarithmic number of comparisons, but they run in linear time. 

- If you pass a sorted range to an algorithm that also takes a comparison function, be sure that the comparison function you pass behaves the same as the one you used to sort the range.

```c++
std::vector<int> v;
std::sort(v.begin(), v.end(), std::greater<int>());

bool a5Exists = std::binary_search(v.begin(), v.end(), 5);                       // error!
bool a5Exists = std::binary_search(v.begin(), v.end(), 5, std::greater<int>());  // right
```

**Item 35: Implement simple case-insensitive string comparisons via `mismatch` or `lexicographical_compare`.**

- Where `strcmp` works only with character arrays, however, `lexicographical_compare` works with ranges of values of any type and it may be passed an arbitrary predicate that determines whether two values satisfy a user-defined criterion.

- `stricmp` / `strcmpi`, being optimized to do exactly one thing, typically run much faster on long strings than do the general-purpose algorithms `mismatch` and `lexicographical_compare`.

```c++
// Using std::mismatch
int ciCharCompare(char c1, char c2) {
  int lc1 = std::tolower(static_cast<unsigned char>(c1));
  int lc2 = std::tolower(static_cast<unsigned char>(c2));

  if (lc1 < lc2) {
    return -1;
  }

  if (lc1 > lc2) {
    return 1;
  }

  return 0;
}

int ciCharCompare(const std::string& s1, const std::string& s2) {
  if (s1.size() < s2.size()) {
    return ciStringCompareImpl(s1, s2);
  } else {
    return -ciStringCompareImpl(s2, s1);
  }
}

int ciStringCompareImpl(const std::string& s1, const std::string& s2) {
  typedef std::pair<std::string::const_iterator, std::string::const_iterator> PSCI;
  PSCI p = std::mismatch(s1.begin(), s1.end(), s2.begin(), std::not2(std::ptr_fun(ciCharCompare)));

  if (p.first == s1.end()) {
    if (p.second == s2.end()) {
      return 0;
    } else {
      return -1;
    }
  }

  return ciCharCompare(*p.first, *p.second);
}

// Using std::lexicographical_compare
bool ciCharLess(char c1, char c2) {
  return std::tolower(static_cast<unsigned char>(c1)) <
         std::tolower(static_cast<unsigned char>(c2));
}

bool ciStringCompare(const std::string& s1, const std::string& s2) {
  return std::lexicographical_compare(s1.begin(), s1.end(), s2.begin(), s2.end(), ciCharLess);
}

// Using stricmp/strcmpi
int ciStringCompare(const std::string& s1, const std::string& s2) {
  return stricmp(s1.c_str(), s2.c_str());
}
```

**Item 36: Understand the proper implementation of `copy_if`.**

- There's a good case to be made for putting `copy_if` into your local STL-related utility library and using it whenever it's appropriate.

```c++
template <typename InputIterator, typename OutputIterator, typename Predicate>
OutputIterator copy_if(InputIterator begin, InputIterator end, OutputIterator destBegin,
                       Predicate p) {
  while (begin != end) {
    if (p(*begin)) {
      *destBegin++ = *begin;
    }
    ++begin;
  }
  return destBegin;
}
```

**Item 37: Use `accumulate` or `for_each` to summarize ranges.**

- This is a probing question why `for_each`'s function parameter is allowed to have side effects while `accumulate`'s is not.

```c++
struct Point {
  Point(double initX, double initY) : x(initX), y(initY) {}
  double x, y;
};

class PointAverage : std::unary_function<Point, void> {
 public:
  PointAverage() : xSum(0), ySum(0), numPoints(0) {}

  void operator()(const Point& p) {
    ++numPoints;
    xSum += p.x;
    ySum += p.y;
  }

  Point result() const { return Point(xSum / numPoints, ySum / numPoints); }

 private:
  std::size_t numPoints;
  double xSum;
  double ySum;
};

std::list<Point> lp;
Point avg = std::for_each(lp.begin(), lp.end(), PointAverage()).result();
```

## CH6: Functions, Functor Classes, Functions, etc.

**Item 38: Design functor classes for pass-by-value.**

- Function pointers are passed by value. STL function objects are modeled after function pointers, so the convention in the STL is that function objects, too, are passed by value (i.e., copied) when passed to and from functions.

- That function pointers are passed by value implies two things.
  - First, your function objects need to be small. Otherwise they will be too expensive to copy.
  - Second, your function objects must be monomorphic (i.e., not polymorphic) -- they must not use virtual functions.

```c++
class Widget;

template <typename T>
class BPFCImpl {
 private:
  Widget w;
  int x;

  virtual ~BPFCImpl();

  virtual void operator()(const T& val) const;

  friend class BPFC<T>;
};

// Functor classes using the "Pimpl Idiom" must support copying in a reasonable fashion.
template <typename T>
class BPFC : public std::unary_function<T, void> {
 private:
  BPFCImpl<T>* pImpl;

 public:
  void operator()(const T& val) const { pImpl->operator(val); }
};
```

**Item 39: Make predicates pure functions.**

- Any place the STL  expects a predicate, it will accept either a real predicate or an object of a predicate class.

- A predicate class is a functor class whose `operator()` function is a predicate, i.e., its `operator()` returns `true` or `false`.

- Declaring `operator()` `const` in predicate classes is necessary for correct behavior, but it's not sufficient. It's also a pure function.

**Item 40: Make functor classes adaptable.**

- Each of the four standard function adapters (`not1`, `not2`, `bind1st`, and `bind2nd`) requires the existence of certain typedefs. Function objects that provide the necessary typedefs are said to be adaptable.

- The conventional way to provide them is to inherit them from a base class, or, more precisely, a base struct (`std::unary_function` or `std::binary_function`).

  - Non-pointer types passed to `unary_function` or `binary_function` have `const`s and references stripped off.
  - Otherwise, pass to `unary_function` or `binary_function` whatever types `operator()` tales pr returns.

```c++
struct WidgetNameCompare : std::binary_function<Widget, Widget, bool> {
  bool operator(const Widget& lhs, const Widget& rhs) const;
};

// The STL implicitly assumes that each functor class has only one operator() function, and it's the
// parameter and return types for this function that should be passed to unary_function or
// bninary_function.
// If you try to combine WidgetNameCompare and PtrWidgetNameCompare, the functor would be adaptable
// with respect to at most one of its calling forms.
struct PtrWidgetNameCompare : std::binary_function<const Widget*, const Widget*, bool> {
  bool operator(const Widget* lhs, const Widget* rhs) const;
};
```

**Item 41: Understand the reasons for `ptr_fun`, `mem_fun`, and `mem_fun_ref`.**

- Functions and function objects are always invoked using the syntactic form for non-member functions.

- You must employ `mem_fun` and `mem_fun_ref` whenever you pass a member function to an STL component, because, in addition to adding typedefs (which may or may not be necessary), they adapt the calling syntaxes from the ones normally used with member functions to the one used everywhere in the STL.

```c++
template <typename InputIterator, typename Function>
Function for_each(InputIterator begin, InputIterator end, Function f) {
  while (begin != end) {
    f(*begin++);
  }
}

// mem_fun takes a pointer to a member function, pmf, and returns an object of type mem_fun_t<R, C>.
// This is a functor class that holds the member function pointer and offers an operator() that
// invokes the pointed-to member function on the object passed to operator().
template <typename R, typename C>
mem_fun_t<R, C> mem_fun(R (C::*pmf)());
```

**Item 42: Make sure `less<T>` means `operator<`.**

- Actually, programmers are allowed to specialize templates in `std` for user-defined types. Almost always, there are alternatives that are superior to specializing `std` templates, but on rare occasions, it's a reasonable thing to do.

- `operator<` is more than just the default way to implement `less`, it's what programmers expect `less` to do.

- If you use `less` (explicitly or implicitly), make sure it means `operator<`. If you want to sort objects using some other criterion, create a special functor class that's not called `less`.

## CH7: Programming with the STL

**Item 43: Prefer algorithm calls to hand-written loops.**

- Efficiency: algorithms are often more efficient than the loops programmers produce.

  - Library implementers can take advantage of their knowledge of container implementations to optimize traversals in a way that no library user ever could.
  - All but the most trivial STL algorithms use computer science algorithms that are more sophisticated -- sometimes much more sophisticated -- than anything the average C++ programmer will be able to come up with.
  - The elimination of redundant computations.

- Correctness: writing loops is more subject to errors than is calling algorithms.

  - One of the trickier things about writing your own loops is making sure you use only iterators that (a) are valid and (b) point where you want them to.

- Maintainability: algorithm calls often yield code that is clearer and more straightforward than the corresponding explicit loops.

  - The names of STL algorithms convey a lot of semantic information, and that makes them clear than any random loop can hope to be.

**Item 44: Prefer member functions to algorithms with the same names.**

- Fist, the member functions are faster.

- Second, they integrate better with the containers (especially the associative containers) than do the algorithms.

  - You get logarithmic-time instead of linear-time performance.
  - You determine whether two values are "the same" using equivalence, which is the natural definition for associative containers.
  - When working with `map`s and `multimap`s, you automatically deal only with key values instead of with (key, value) pairs.
  - Each of the algorithms that `list` specializes (`remove`, `remove_if`, `unique`, `sort`, `merge`, and `reverse`) copies objects, but `list`-specific versions copy nothing; they simply manipulate the pointers connecting list nodes.

**Item 45: Distinguish among `count`, `find`, `binary_search`, `lower_bound`, `upper_bound`, and `equal_range`.**

| What you want to know                                        | Algorithm to Use     |                                | Member Function to Use |                                 |
| ------------------------------------------------------------ | -------------------- | ------------------------------ | ---------------------- | ------------------------------- |
|                                                              | On an Unsorted Range | On  a Sorted Range             | With a `set` or `map`  | With a `multiset` or `multimap` |
| Does the desired value exist?                                | `find`               | `binary_search`                | `count`                | `find`                          |
| Does the desired value exist? If so, where is the first object with that value? | `find`               | `equal_range`                  | `find`                 | `find` or `lower_bound`         |
| Where is the first object with a value not preceding the desired value? | `find_if`            | `lower_bound`                  | `lower_bound`          | `lower_bound`                   |
| Where is the first object with a value succeeding the desired value? | `find_if`            | `upper_bound`                  | `upper_bound`          | `upper_bound`                   |
| How many objects have the desired value?                     | `count`              | `equal_range`, then `distance` | `count`                | `count`                         |
| Where are all the objects with the desired value?            | `find` (iteratively) | `equal_range`                  | `equal_range`          | `equal_range`                   |

**Item 46: Consider function objects instead of functions as algorithm parameters.**

- Function objects as parameters to algorithms offer more than greater efficiency. They're also more robust when it comes to getting your code to compile.

```c++
// Advantage 1: inline.
std::vector<double> v;

// An indirect function call through a pointer and most compilers won't try to inline calls to functions that are invoked through function pointers
inline bool doubleGreater(double d1, double d2) { return d1 > d2; }
std::sort(v.begin(), v.end(), doubleGreater);

// Compilers are able to perform optimizations on this call-free code that are otherwise not usually attempted.
std::sort(v.begin(), v.end(), std::greater<double>());

// Advantage 2: not uncommon for STL platforms to reject perfectly valid code.
std::set<std::string> s;

// The cause of the problem is that this particular STL platform has a bug in its handling of const member functions.
std::transform(s.begin(), s.end(), std::ostream_iterator<std::string::size_type>(std::cout, "\n"),
               std::mem_fun_ref(&std::string::size));

// It also facilities inlining the call to string::size.
struct StringSize : public std::unary_function<std::string, std::string::size_type> {
  std::string::size_type operator()(const std::string& s) const { return s.size(); }
};
std::transform(s.begin(), s.end(), std::ostream_iterator<std::string::size_type>(std::cout, "\n"),
               StringSize());

// Advantage 3: avoid subtle language pitfalls.
template <typename FPType>
FPType average(FPType val1, FPType val2) {
  return (val1 + val2) / 2;
}

// In theory, there could be another function template named average that takes a single type parameter.
template <typename InputIterator1, typename InputIterator2>
void writeAverage(InputIterator1 begin1, InputIterator1 end1, InputIterator2 begin2,
                  std::ofstream& s) {
  std::transform(
      begin1, end1, begin2,
      std::ostream_iterator<typename std::iterator_traits<InputIterator1>::value_type>(s, "\n"),
      average<typename std::iterator_traits<InputIterator1>::value_type>);
};
```

**Item 47: Avoid producing write-only code.**

- As you write the code, it seems straightforward, because it's a natural outgrowth of some basic ideas. Readers, however, have great difficulty in decomposing the final product back into the ideas on which it is based. That's the calling card of write-only code: it's easy to write, but it's hard to read and understand.

**Item 48: Always `#include` the proper headers.**

- Any time you use any of the components in a header, be sure to provide the corresponding `#include` directive, even if your development platform lets you get away without it.

**Item 49: Learn to decipher STL-related compiler diagnostics.**

- For `vector` and `string`, iterators are sometimes pointers, so compiler diagnostics may refer to pointer types if you've made a mistake with an `iterator`.

- Messages mentioning `back_insert_iterator`, `front_insert_iterator`, or `insert_iterator` almost always mean you've made a mistake calling `back_inserter`, `front_inserter`, or `inserter`, respectively. If you didn't call these functions, some function you called (directly or indirectly) did.

- If you get a message mentioning `binder1st` or `binder2nd`, you've probably made a mistake using `bind1st` or `bind2nd`.

- Output iterators do their outputting or inserting work inside assignment operators, so if you've made a mistake with one of these iterator types, you're likely to get a message complaining about something inside an assignment operator you've never heard of.

- If you get an error message originating from inside the implementation of an STL algorithm, there's probably something wrong with the types you're trying to use with that algorithm.

- If you're using a common STL component like `vector`, `string`, or the `for_each` algorithm, and a compiler says it has no idea what you're talking about, you've probably failed to `#include` a required header file.

**Item 50: Familiarize yourself with STL-related web sites.**

- SGI's library implementation goes beyond the STL. Their goal is the development of a complete implementation of the standard C++ library, except for the parts inherited from C.

- STLport offers a modified version of SGI's STL implementation that's been ported to more than 20 compilers. It offers a "debug mode" to help detect improper use of the STL -- uses that compile but lead to undefined runtime behavior.

- Boost offers itself as a vetting mechanism to help separate the sheep from the goats when it comes to potential additions to the standard C++ library.
