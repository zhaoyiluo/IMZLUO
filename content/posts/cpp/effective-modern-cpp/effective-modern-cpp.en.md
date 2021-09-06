---
title: "[NOTE] Effective Modern C++"
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

- During template type deduction, arguments that are references are treated as non-references, i.e., their reference-ness is ignored.

- When deducing types for universal reference parameters, lvalue arguments get special treatment.

- When deducing types for by-value parameters, `const` and/or `volatile` arguments are treated as non-`const` and non-`volatile`.

- During template type deduction, arguments that are array or function names decay to pointers, unless they're used to initialize references.

**Item 2: Understand `auto` type deduction.**

- `auto` type deduction is usually the same as template type deduction, but `auto` type deduction assumes that a braced initializer represents a `std::initializer_list`, and template type deduction doesn't.

- `auto` in a function return type or a lambda parameter implies template type deduction, not `auto` type deduction.

```c++
// wrong usage 1
auto createInitList() { return {1, 2, 3}; }

// wrong usage 2
std::vector<int> v;
auto resetV = [&v](const auto& newValue) { v = newValue; };
resetV({1, 2, 3});

// right usage
template <typename T>
void f(std::initializer_list<T> initList);
f({11, 23, 9});
```

**Item 3: Understand `decltype`.**

- `decltype` almost always yields the type of a variable or expression without any modifications.

- For lvalue expressions of type `T` other than names, `decltype` always reports a type of `T&`.

- C++14 supports `decltype(auto)`, which, like `auto`, deduces a type from its initializer, but it performs the type deduction using the `decltype` rules.

```c++
// C++14 version
template <typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i) {
  return std::forward<Container>(c)[i];  // apply std::forward to universal references
}

// C++11 version
template <typename Container, typename Index>
auto authAndAccess(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i]) {
  return std::forward<Container>(c)[i];  // apply std::forward to universal references
}

decltype(auto) f1() {
  int x = 0;
  return x;  // decltype(x) is int, so f1 returns int
}

decltype(auto) f1() {
  int x = 0;
  return (x);  // decltype((x)) is int&, so f2 returns int&
}
```

**Item 4: Know how to view deduced types.**

- Deduced types can often be seen using IDE editors, compiler error messages, and the Boost TypeIndex library.

- The results of some tools may be neither helpful nor accurate, so an understanding of C++'s type deduction rules remains essential.

  - The specification for `std::type_info::name` mandates that the type be treated as if it had been passed to a template function as a by-value parameter.

## CH2: `auto`

**Item 5: Prefer `auto` to explicitly type declarations.**

- `auto` variables must be initialized, are generally immune to type mismatches that can lead to portability or efficiency problems, can ease the process of refactoring, and typically require less typing than variables with explicitly specified types.

- `auto`-typed variables are subject to the pitfalls described in Items 2 and 6.

**Item 6: Use the explicitly typed initializer idiom when `auto` deduces undesired types.**

- "Invisible" proxy types can cause `auto` to deduce the "wrong" type for an initializing expression.

- The explicitly typed initializer idiom forces `auto` to deduce the type you want it to have.

```c++
class Widget;
std::vector<bool> features(const Widget& w);
void processWidget(Widget& w, bool priority);

Widget w;

// undefined behavior
// the call to features returns a temporary std::vector<bool> object and operator[] returns
// std::vector<bool>::reference object, which contains a pointer to a word plus the offset
// corresponding to bit 5; highPriority is a copy of the temporary reference object and also
// contains a pointer to a word in the above temporary reference object; at the end of the
// statement, this pointer becomes a dangling pointer
auto highPriority = features(w)[5];
processWidget(w, highPriority);

// the explicitly typed initializer idiom
auto highPriority = static_cast<bool>(features(w)[5]);
processWidget(w, highPriority);
```

## CH3: Moving to Modern C++

**Item 7: Distinguish between `()` and `{}` when creating objects.**

- Braced initialization is the most widely usable initialization syntax, it prevents narrowing conversions, and it's immune to C++'s most vexing parse.

- During constructor overload resolution, braced initializers are matched to `std::initializer_list` parameters if at all possible, even if other constructors offer seemingly better matches.

- An example of where the choice between parentheses and braces can make a significant difference is creating a `std::vector<numeric type>` with two arguments.

- Choosing between parentheses and braces for object creation inside templates can be challenging.

```c++
// uncopyable objects may be initialized using braces or parentheses, but not using "="
std::atomic<int> ai1{0};   // fine
std::atomic<int> ai2(0);   // fine
std::atomic<int> ai3 = 0;  // error!

// prohibit implicit narrowing conversions among built-in types
double x, y, z;
int sum1{x + y + z};  // error!

// not possible to know which should be used if you're a template author
template <typename T, typename... TS>
void doSomeWork(Ts&&... params) {
  // create local T object from params ...
}

T localObject(std::forward<Ts>(params)...);  // using parens
T localObject{std::forward<Ts>(params)...};  // using braces

std::vector<int> v;
doSomeWork<std::vector<int>>(10, 20);
```

**Item 8: Prefer `nullptr` to `0` and `NULL`.**

- Prefer `nullptr` to `0` and `NULL`.

- Avoid overloading on integral and pointer types.

```c++
class Widget;
std::mutex f1m, f2m, f3m;

int f1(std::shared_ptr<Widget> spw);
int f2(std::unique_ptr<Widget> upw);
bool f3(Widget* pw);

template <typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr) {
  using MuxGuard = std::lock_guard(MuxType);
  MuxGuard g(mutex);
  return func(ptr);
}

auto result1 = lockAndCall(f1, f1m, 0);        // error!
auto result2 = lockAndCall(f2, f2m, NULL);     // error!
auto result3 = lockAndCall(f3, f3m, nullptr);  // fine
```

**Item 9: Prefer alias declarations to `typedef`s.**

- `typedef`s don't support templatization, but alias declarations do.

- Alias templates avoid the "`::type`" suffix and, in templates, the "`typename`" prefix often required to refer to `typedef`s.

- C++14 offers alias templates for all the C++11 type traits transformations.

```c++
class Widget;

// use typedefs
template <typename T>
struct MyAllocList {
  typedef std::list<T, MyAlloc<T>> type;
};
MyAllocList<Widget>::type lw;

template <typename T>
class Widget {
 private:
  typename MyAllocList<T>::type list;
};

// use alias
template <typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;
MyAllocList<Widget> lw;

template <typename T>
class Widget {
 private:
  MyAllocList<T> list;  // no "typename", no "::type"
};
```

**Item 10: Prefer scoped `enum`s to unscoped `enum`s.**

- C++98-style `enum`s are now known as unscoped `enum`s.

- Enumerators of scoped `enum`s are visible only within the `enum`. They convert to other types only with a cast.

- Both scoped and unscoped `enum`s support specification of the underlying type. The default underlying type for scoped `enum`s is `int`. Unscoped `enum`s have no default underlying type.

- Scoped `enum`s may always be forward-declared. Unscoped `enum`s may be forward-declared only if their declaration specifies an underlying type.

```c++
using UserInfo = std::tuple<std::string, std::string, std::size_t>;

// useful for unscoped enums
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
auto val = std::get<uiEmail>(uInfo);

// pay for scoped enums
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);

// std::get is a template and its argument has to be known during compilation
// the generalized function should be a constexpr function
template <typename E>
constexpr typename std::underlying_type<E>::type toUType(E enumerator) noexcept {  // C++11
  return static_cast<typename std::underlying_type<E>::type>(enumerator);
}
template <typename E>
constexpr auto toUType(E enumerator) noexcept {  // C++14
  return static_cast<std::underlying_type_t<E>>(enumerator);
}
```

**Item 11: Prefer deleted functions to private undefined ones.**

- Prefer deleted functions to private undefined ones.

- Any function may be deleted, including non-member functions and template instantiations.

```c++
class Widget {
 public:
  template <typename T>
  void processPointer(T* ptr) {
    // ...
  }

 private:
  template <>  // error!
  void processPointer<void>(void*);
};
// template specializations must be written at namespace scope, not class scope
template <>
void Widget::processPointer<void>(void*) = delete;  // fine
```

**Item 12: Declare overriding functions `override`.**

- Declare overriding functions `override`.

- Member function reference qualifiers make it possible to treat lvalue and rvalue objects (`*this`) differently.

**Item 13: Prefer `const_iterator`s to `iterator`s.**

- Prefer `const_iterator`s to `iterator`s.

- In maximally generic code, prefer non-member versions of `begin`, `end`, `rbegin`, etc., over their member function counterparts.

```c++
// invoking the non-member begin function on a const container yields a const_iterator and that
// iterator is what this template returns
template <class C>
auto cbegin(const C& container) -> decltype(std::begin(container)) {
  return std::begin(container);
}
```

**Item 14: Declare functions `noexcept` if they won't emit exceptions.**

- `noexcept` is part of a function's interface, and that means that callers may depend on it.

- `noexcept` functions are more optimizable than non-`noexcept` functions.

  - Optimizers need not keep the runtime stack in an unwindable state if an exception would propagate out of the function, nor must they ensure that objects in a `noexcept` function are destroyed in the inverse oder of construction should an exception leave the function.

- `noexcept` is particularly valuable for the move operations, `swap`, memory deallocation functions, and destructors.

  - `swap` functions are conditionally `noexcept` and they depend on whether the expressions inside the `noexcept` clauses are `noexcept`.
  - The only time a destructor is not implicitly `noexcept` is when a data member of the class is of a type that expressly states that its destructor may emit exceptions.

- Most functions are exception-neutral rather than `noexcept`.

**Item 15: Use `constexpr` whenever possible.**

- `constexpr` objects are `const` and are initialized with values known during compilation.

- `constexpr` functions can produce compile-time results when called with arguments whose values are known during compilation.

  - `constexpr` functions are limited to taking and returning literal types.
  - In C++11, `constexpr` functions may contain no more than a single executable statement: a `return`.
  - In C++11, all built-in types except `void` qualify, but user-defined types may be literal, too.

- `constexpr` objects and functions may be used in a wider range of contexts than non-`constexpr` objects and functions.

- `constexpr` is part of an object's or function's interface.

```c++
class Point {
 public:
  constexpr Point(double xVal = 0, double yVal = 0) noexcept : x(xVal), y(yVal) {}

  constexpr double xValue() const noexcept { return x; }
  constexpr double yValue() const noexcept { return y; }

  // C++11
  void setX(double newX) noexcept { x = newX; }
  void setY(double newY) noexcept { y = newY; }

  // C++14
  constexpr void setX(double newX) noexcept { x = newX; }
  constexpr void setY(double newY) noexcept { y = newY; }

 private:
  double x, y;
};

constexpr Point midpoint(const Point& p1, const Point& p2) noexcept {
  return {(p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2};
}

constexpr Point reflection(const Point& p) noexcept {
  Point result;

  // C++14 required
  result.setX(-p.xValue());
  result.setY(-p.yValue());

  return result;
}

constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflected = reflection(mid);  // C++14 required
```

**Item 16: Make `const` member functions thread safe.**

- Make `const` member functions thread safe unless you're certain they'll never be used in a concurrent context.

- Use of `std::atomic` variables may offer better performance than a mutex, but they're suited for manipulation of only a single variable or memory location.

**Item 17: Understand special member function generation.**

- The special member functions are those compilers may generate on their own: default constructor, destructor, copy operations, and move operations.

- Move operations are generated only for classes lacking explicitly declared move operations, copy operations, and a destructor.

- The copy constructor is generated only for classes lacking an explicitly declared copy constructor, and it's deleted if a move operation is declared. The copy assignment operator is generated only for classes lacking an explicitly declared copy assignment operator, and it's deleted if a move operation is declared. Generation of the copy operations in classes with an explicitly declared copy operation or destructor is deprecated.

- Member function templates never suppress generation of special member functions.

```c++
class Base {
 public:
  virtual ~Base() = default;

  Base(Base&&) = default;  // support moving
  Base& operator=(Base&&) = default;

  Base(const Base&) = default;  // support copying
  Base& operator=(const Base&) = default;

  // ...
};
```

## CH4: Smart Pointers

**Item 18: Use `std::unique_ptr` for exclusive-ownership resource management.**

- `std::unique_ptr` is a small, fast, move-only smart pointer for managing resources with exclusive-ownership semantics.

- By default, resource destruction takes place via `delete`, but custom deleters can be specified. Stateful deleters and function pointers as deleters increase the size of `std::unique_ptr` objects.

- Attempting to assign a raw pointer (e.g., from `new`) to `std::unique_ptr` won't compile, because it would constitute an implicit conversion from a raw to a smart pointer. Such implicit conversions can be problematic, so C++11's smart pointers prohibit them.

- Converting a `std::unique_ptr` to a `std::shared_ptr` is easy.

**Item 19: Use `std::shared_ptr` for shared-ownership resource management.**

- `std::shared_ptr`s offer convenience approaching that of garbage collection for the shared lifetime management of arbitrary resources.

- Compared to `std::unique_ptr`, `std::shared_ptr` objects are typically twice as big, incur overhead for control blocks, and require atomic reference count manipulations.

  - `std::make_shared` always creates a control block.
  - A control block is created when a `std::shared_ptr` is constructed from a unique-ownership pointer.
  - When a `std::shared_ptr` constructor is called with a raw pointer, it creates a control block.

- Default resource destruction is via `delete`, but custom deleters are supported. They type of the deleter has no effect on the type of the `std::shared_ptr`.

- Avoid creating `std::shared_ptr`s from variables of raw pointer type.

```c++
// a data structure keeps track of Widgets that have been processed
class Widget;
std::vector<std::shared_ptr<Widget>> processedWidgets;

// bad example
class Widget {
 public:
  // ...
  void process();
  // ...
};

// this is wrong, though it will compile
// std::shared_ptr will create a new control block for the pointed-to Widget (*this)
void Widget::process() {
  // ...
  processedWidgets.emplace_back(this);
}

// good example
class Widget : public std::enable_shared_from_this<Widget> {
 public:
  // factory function that perfect-forwards args to a private ctor
  template <typename... Ts>
  static std::shared_ptr<Widget> create(Ts&&... params);

  void process();
  // ...

 private:
  // ctors
};

// shared_from_this creates a std::shared_ptr to the current object, but it does it without
// duplicating control blocks
// the design relies on the current object having an associated control block, thus factory
// function is applied
void Widget::process() {
  // ...
  processedWidgets.emplace_back(shared_from_this());
}
```

**Item 20: Use `std::weak_ptr` for `std::shared_ptr`-like pointers that can dangle.**

- Use `std::weak_ptr` for `std::shared_ptr`-like pointers that can dangle.

- Potential use cases for `std::weak_ptr` include caching, observer lists, and the prevention of `std::shared_ptr` cycles.

```c++
class Widget;
class WidgetID;

std::unique_ptr<const Widget> loadWidget(WidgetID id);

// the Observer design pattern
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id) {
  static std::unordered_map<WidgetID, std::weak_ptr<const Widget>> cache;
  auto objPtr = cache[id].lock();
  if (!objPtr) {
    objPtr = loadWidget(id);
    cache[id] = objPtr;
  }
  return objPtr;
}
```

**Item 21: Prefer `std::make_unique` and `std::make_shared` to direct use of `new`.**

- Compared to direct use of `new`, `make` functions eliminate source code duplication, improve exception safety, and, for `std::make_shared` and `std::allocate_shared`, generate code that's smaller and faster.

- Situations where use of `make` functions is inappropriate include the need to specify custom deleters and a desire to pass braced initializers.

- For `std::shared_ptr`s, additional situations where `make` functions may be ill-advised include (1) classes with custom memory management and (2) systems with memory concerns, very large objects, and `std::weak_ptr`s that outlive the corresponding `std::shared_ptr`s.

```c++
class Widget;

// advantage 1 (the same applies to std::shared_ptr)
auto upw1(std::make_unique<Widget>());
std::unique_ptr<Widget> upw2(new Widget);

// advantage 2 (the same applies to std::shared_ptr)
void processWidget(std::unique_ptr<Widget> upw, int priority);
int computePriority();
processWidget(std::unique_ptr<Widget>(new Widget), computePriority());  // potential resource leak!
processWidget(std::make_unique<Widget>(), computePriority());  // no potential resource leak

// advantage 3 (for std::shared_ptr only)
std::shared_ptr<Widget> spw(new Widget);  // two allocations for Widget and control block
auto spw = std::make_shared<Widget>();    // one allocation

// disadvantage 1 (the same applies to std::shared_ptr)
auto widgetDeleter = [](Widget* pw) {};
std::unique_ptr<Widget, decltype(widgetDeleter)> upw(new Widget, widgetDeleter);

// disadvantage 2 (the same applies to std::shared_ptr)
auto upv = std::make_unique<std::vector<int>>(10, 20);  // apply perfect-forward and use parentheses
auto initList = {10, 20};
auto upv = std::make_unique<std::vector<int>>(initList);
```

**Item 22: When using the Pimpl Idiom, define special member functions in the implementation file.**

- The Pimpl Idiom decreases build times by reducing compilation dependencies between class clients and class implementations.

- For std::unique_ptr pImpl pointers, declare special member functions in the class header, but implement them in the implementation file. Do this even if the default function implementations are acceptable.

- The above advice applies to `std::unique_ptr`, but not to `std::shared_ptr`.

  - For `std::shared_ptr`, the type of the deleter is not part of the type of the smart pointer. This necessitates larger runtime data structures and somewhat slower code, but pointed-to types need not be complete when compiler-generated special functions are employed.

```c++
// "widget.h"
class Widget {
 public:
  Widget();
  ~Widget();
  Widget(Widget&& rhs) noexcept;
  Widget& operator=(Widget& rhs) noexcept;
  // ...

 private:
  struct Impl;
  std::unique_ptr<Impl> pImpl;
};

// "widget.cpp"
#include <string>
#include <vector>
#include "gadget.h"
#include "widget.h"

struct Widget::Impl {
  std::string name;
  std::vector<double> data;
  Gadget g1, g2, g3;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()) {}

// (if no definition)
// prior to using delete, however, implementations typically have the default deleter employ C++11's
// static_assert to ensure that the raw pointer doesn't point to an incomplete type; when the
// compiler generates code for the destruction of the Widget w, then, it generally encounters a
// statc_assert that fails, and that's usually what leads to the error message; this message is
// associated with the point where w is destroyed, because Widget's destructor, like all
// compiler-generated special member functions, is implicitly inline; the message itself often
// refers to the line where w is created, because it's the source code explicitly creating the
// object that leads to its later implicit destruction
Widget::~Widget() = default;

// (if no definition)
// the problem here is that compilers must be able to generate code to destroy pImpl in the event
// that an exception arises inside the move constructor (even if the constructor is noexcept!), and
// destroying pImpl requires that Impl be complete
Widget::Widget(Widget&& rhs) noexcept = default;

// (if no definition)
// the compiler-generated move assignment operator needs to destroy the object pointed to by pImpl
// before reassigning it, but in the Widget header file, pImpl points to an incomplete type
Widget& Widget::operator=(Widget&& rhs) noexcept = default;
```

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