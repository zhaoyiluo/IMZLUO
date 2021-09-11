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

- `std::move` performs an unconditional cast to an rvalue. In and of itself, it doesn't move anything.

- `std::forward` casts its argument to an rvalue only if that argument is bound to an rvalue.

- Neither `std::move` or `std::forward` do anything at runtime.

- Move requests on `const` objects are treated as copy requests.

**Item 24: Distinguish universal references from rvalue references.**

- If a function template parameter has type `TT&` for a deduced type `T`, or if an object is declared using `auto&&`, the parameter or object is a universal reference.

- If the form of the type declaration isn't precisely `type&&`, or if type deduction does not occur, `type&&` denotes an rvalue reference.

- Universal references correspond to rvalue references if they're initialized with rvalues. They correspond to lvalue references if they're initialized with lvalues.

```C++
class Widget;

void f(Widget&& param);  // rvalue reference

Widget&& var1 = Widget();  // rvalue reference

auto&& var2 = var1;  // universal reference

template <typename T>
void f(std::vector<T>&& param);  // rvalue reference

template <typename T>
void f(T&& param);  // universal reference

template <class T, class Allocator = std::allocator<T>>
class vector {
 public:
  // rvalue reference
  // there's no type deduction in this case because push_back can't exist without a particular
  // vector instantiation for it to be part of, and the type of that instantiation fully determines
  // the declaration for push_back
  void push_back(T&& x);

  // the type parameter Args is independent of vector's type parameter T, so Args must be deduced
  // each time emplace_back is called
  template <class Args>
  void emplace_back(Args&&... args);

  // ...
};
```

**Item 25: Use `std::move` on rvalue references, `std::forward` on universal references.**

- Apply `std::move` to rvalue references and `std::forward` to universal references the last time each is used.

- Do the same thing for rvalue references and universal references being returned from functions that return by value.

- Never apply `std::move` or `std::forward` to local objects if they would otherwise be eligible for the return value optimization.

  - The compilers may elide the copying (or moving) of a local object in a function that returns by value if (1) the type of the local object is the same as that returned by the function and (2) the local object is what's being returned.
  - If the conditions for the RVO are met, but compilers choose not to perform copy elision, the object being returned must be treated as an rvalue.

```c++
// in a function that returns by value and returns an object bound to an rvalue reference or a
// universal reference, apply std::move or std::forward when you return the reference
Matrix operator+(Matrix&& lhs, const Matrix& rhs) {
  lhs += rhs;
  return std::std::move(lhs);  // move lhs into return value
}

template <typename T>
Fraction reduceAndCopy(T&& frac) {
  frac.reduce();
  return std::forward<T>(frac);
}
```

**Item 26: Avoid overloading on universal references.**

- Overloading on universal references almost always leads to the universal reference overload being called more frequently than expceted.

- Perfect-forwarding constructors are especially problematic, because they're typically better matches than copy constructors for non-`const` lvalues, and they can hijack derived class calls to base class copy and move constructors.

```c++
std::string nameFromIdx(int idx);

class Person {
 public:
  template <typename T>
  explicit Person(T&& n) : name(std::forward<T>(n)) {}

  explicit Person(int idx) : name(nameFromIdx(idx)) {}

  Person(const Person& rhs);  // compiler-generated

  Person(Person&& rhs);  // compiler-generated

 private:
  std::string name;
};

// compiler is being initialized with a non-const lvalue (p), and that means that the templatized
// constructor can be instantiated to take a non-const lvalue of type Person
// explicit Person(Person& n) : name(std::forward<Person>(n)) {}
// calling the copy constructor would require adding const to p to match the copy constructor's
// parameter's type
Person p("Nancy");
auto cleanOfP(p);  // this won't compile!

// situations where a template instantiation and a non-template function are equally good matches
// for a function call, the normal function is preferred
const Person p("Nancy");
auto cleanOfP(p);  // call copy constructor

// these two constructors will call base class forwarding constructor because the derived class
// functions are using arguments of type SpecialPerson to pass to their base class
class SpecialPerson : public Person {
 public:
  SpecialPerson(const SpecialPerson& rhs) : Person(rhs) {}
  SpecialPerson(SpecialPerson&& rhs) : Person(std::move(rhs)) {}
};
```

**Item 27: Familiarize yourself with alternatives to overloading on universal references.**

- Alternatives to the combination of universal references and overloading include the use of distinct function names, passing parameters by lvalue-reference-to-`const`, passing parameters by value, and using tag dispatch.

- Constraining templates via `std::enable_if` permits the use of universal references and overloading together, but it controls the conditions under which compilers may use the universal reference overloads.

- Universal reference parameters often have efficiency advantages, but they typically have usability disadvantages.

```c++
// use tag dispatch
template <typename T>
void logAndAdd(T&& name) {
  logAndAddImpl(std::forward<T>(name), std::is_integral<std::remove_reference_t<T>::type>());
}

template <typename T>
void logAndAddImpl(T&& name, std::false_type) {
  auto now = std::chrono::system_clock::now();
  log(now, "logAndAdd");
  names.emplace(std::forward<T>(name));
}

void logAndAddImpl(int idx, std::true_type) { logAndAdd(nameFromIdx(idx)); }

// constrain templates that take universal references
class Person {
 public:
  template <typename T,
            typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value &&
                                        !std::is_integral<std::remove_reference_t<T>>::value>>
  explicit Person(T&& n) : name(std::forward<T>(n)) {
    // assert that a std::string can be created from a T object
    static_assert(std::is_constructible<std::string, T>::value,
                  "Parameter n can't be used to construct a std::string");

    // the usual constructor work goes here
  }

  // remainder of Person class
};
```

**Item 28: Understand reference collapsing.**

- Reference collapsing occurs in four contexts: template instantiation, `auto` type generation, creation and use of `typedef`s and alias declarations, and `decltype`.

- When compilers generate a reference to a reference in a reference collapsing context, the result becomes a single reference. If either the original references is an lvalue reference, the result is an lvalue reference. Otherwise it's an rvalue reference.

- Universal references are rvalue references in contexts where type deduction distinguishes lvalues from rvalues and where reference collapsing occurs.

```c++
template <typename T>
T&& forward(std::remove_reference_t<T>& param) {
  return static_assert<T&&>(param);
}
```

**Item 29: Assume that move operations are not present, not cheap, and not used.**

- Assume that move operations are not present, not cheap, and not used.

  - No move operations: The object to be moved from fails to offer move operations. The move request therefore becomes a copy request.
  - Move not fast: The object to be moved from has move operations that are no faster than its copy operations.
  - Move not usable: The context in which the moving would take place requires a move operation that emits no exceptions, but that operation isn't declared `noexcept`.
  - Source object is lvalue: With very few exceptions only rvalues may be used as the source of a move operation.

- In code with known types or support for move semantics, there is no need for assumptions.

**Item 30: Familiarize yourself with perfect forwarding failure cases.**

- Perfect forwarding fails when template type deduction fails or when it deduces the wrong type.

- The kinds of arguments that lead to perfect forwarding failure are braced initializers, null pointers expressed as `0` or `NULL`, declaration-only integral `const static` data members, template and overloaded function names, and bitfields.

```c++
template <typename... Ts>
void fwd(Ts&&... params) {
  f(std::forward<Ts>(params)...);
}

// failure case 1: braced initializers
// the problem is that passing a braced initializer to a function template parameter that's not
// declared to be a std::initializer_list is decreed to be, as the Standard puts it, a "non-deduced
// context"
void f(const std::vector<int>& v);
f({1, 2, 3});  // fine

fwd({1, 2, 3});  // error!

auto il = {1, 2, 3};
fwd(il);  // fine

// failure case 2: 0 or NULL as null pointers
// the problem is that neither 0 or NULL can be perfect-forwarded as a null pointer

// failure case 3: declaration-only integral static const and constexpr data members
// the problem is that compilers perform const propagation on such members' values, thus eliminating
// the need to set aside memory for them and references are simply pointers that are automatically
// dereferenced
class Widget {
 public:
  static constexpr std::size_t MinVals = 28;
  // ...
};

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);  // error！ shouldn't link

constexpr std::size_t Widget::MinVals;  // better to provide a definition

// failure case 4: overloaded function names and template names
// the problem is that a function template doesn't represent one function, it represents many
// functions
void f(int (*pf)(int));
void f(int pf(int));

int processVal(int value);
int processVal(int value, int priority);

f(processVal);   // fine
fwd(processVal)  // error! which processVal?

    template <typename T>
    T workOnVal(T param) {}

fwd(workOnVal);  // error! which workOnVal instantiation?

using ProcessFuncType = int (*)(int);
ProcessFuncType processValPtr = processVal;
fwd(processValPtr);                          // fine
fwd(static_cast<processValPtr>(workOnVal));  // also fine

// failure case 5: bitfields
// the problem is that a non-const reference shall not be bound to a bit-field and
// reference-to-const don't bind to bitfields, they bind to "normal" objects (e.g., int) into which
// the values of the bitfields have been copied
struct IPv4Header {
  std::uint32_t version : 4, IHL : 4, DSCP : 6, ECN : 2, totalLength : 16;
};

void f(std::size_t sz);

IPv4Header h;
f(h.totalLength);    // fine
fwd(h.totalLength);  // error!

auto length = static_cast<std::uint16_t>(h.totalLength);
fwd(length);  // fine
```

## CH6: Lambda Expressions

**Item 31: Avoid default capture modes.**

- Default by-reference capture can lead to dangling references.

- Default by-value capture is susceptible to dangling pointers (especially `this`), and it misleadingly suggests that lambdas are self-contained.

- Lambdas may also be dependent on objects with static storage duration that are defined at global or namespace scope or are declared `static` inside classes, functions, or files. These objects can be used inside lambdas, but they can't be captured.

```c++
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;

// by-reference capture issue
void addDivisorFilter() {
  auto calc1 = computeSomeValue1();
  auto calc2 = computeSomeValue2();

  auto divisor = computeDivisor(calc1, calc2);

  filters.emplace_back([&](int value) { return value % divisor == 0; });
}

// by-value capture issue
class Widget {
  void addFilter() const;

 private:
  int divisor;
};

// captures apply only to non-static local variables (including parameters) visible in the scope
// where the lambda is created
void Widget::addFilter() const {
  filters.emplace_back([=](int value) { return value % divisor == 0; });
}
void Widget::addFilter() const {
  auto currentObjectPtr = this;
  filters.emplace_back(
      [currentObjectPtr](int value) { return value % currentObjectPtr->divisor == 0; });
}

void doSomeWork() {
  auto pw = std::make_unique<Widget>();
  pw->addFilter();  // undefined behavior when funtion returned
}

// solution: use generalized lambda capture (C++14)
void Widget::addFilter() const {
  filters.emplace_back([divisor = divisor](int value) { return value % divisor == 0; });
}
```

**Item 32: Use init capture to move objects into closures.**

- Use C++14's init capture to move objects into closures.

- In C++11, emulate init capture via hand-written classes or `std::bind`.

  - It's not possible to move=construct an object into a C++1 closure, but it is possible to move-construct an object into a C++11 bind object.
  - Emulating move-capture in C++11 consists of move-constructing an object into a bind object, then passing the move-constructed object to the lambda by reference.
  - Because the lifetime of the bind object is the same as that of the closure, it's possible to treat objects in the bind object as if they were in the closure.

```c++
class Widget {
 public:
  // ...
  bool isValidated() const;
  bool isProcessed() const;
  bool isArchived() const;

 private:
  // ...
};

// C++11
auto func = std::bind(
    [](const std::unique_ptr<Widget>& pw) { return pw->isValidated() && pw->isArchived(); },
    std::make_unique<Widget>());

// C++14
auto func = [pw = std::make_unique<Widget>()] { return pw->isValidated() && pw->isArchived(); };
```

**Item 33: Use `decltype` on `auto&&` parameters to `std::forward` them.**

- Use `decltype` on `auto&&` parameters to `std::forward` them.

```c++
// the only thing the lambda does with its parameter x is forward it to normalize
auto f = [](auto x) { return normalize(x); };

class SomeCompilerGeneratedClassName {
 public:
  template <typename T>
  auto operator()(T x) const {
    return normalize(x);
  }
};

// more elegant style
auto f = [](auto&& x) { return normalize(std::forward<decltype(x)>(x)); };

// more more elegant style
auto f = [](auto&&... xs) { return normalize(std::forward<decltype(xs)>(xs)...); };
```

**Item 34: Prefer lambdas to `std::bind`.**

- Lambdas are more readable, more expressive, and may be more efficient than using `std::bind`.

- In C++11 only, `std::bind` may be useful for implementing move capture or for binding objects with templatized function call operators.

- `std::bind` always copies its arguments, but callers can achieve the effect of having an argument stored by reference by applying `std::ref` to it. All arguments passed to bind objects are passed by reference, because the function call operator for such objects uses perfect forwarding.

```c++
// polymorphic function objects
class PolyWidget {
 public:
  template <typename T>
  void operator()(const T& param) const;
  // ...
};

PolyWidget pw;
auto boundPW = std::bind(pw, std::placeholders::_1);

boundPW(1930);
boundPW(nullptr);
boundPW("Rosebud");
```

## CH7: The Concurrency API

**Item 35: Prefer task-based programming to thread-based.**

- The `std::thread` API offers no direct way to get return values from asynchronously run functions, and if those functions throw, the program is terminated.

- Thread-based programming calls for manual management of thread exhaustion, oversubscription, load balancing, and adaptation to new platforms.

- Task-based programming via `std::async` with the default launch policy handles most of these issues for you.

- There are some situations where using threads directly may be appropriate, though these are uncommon cases.

  - You need access to the API of the underlying threading implementation.
  - You need to and are able to optimize thread usage for your application.
  - You need to implement threading technology beyond the C++ concurrency API.

**Item 36: Specify `std::launch::async` if asynchronicity is essential.**

- The default launch policy for `std::async` permits both asynchronous and synchronous task execution.

- This flexibility leads to uncertainty when accessing `thread_local`s, implies that the task may never execute, and affects program logic for timeout-based `wait` calls.

- Specify `std::launch::async` if asynchronous task execution is essential.

```c++
using namespace std::literals;

void f() { std::this_thread::sleep_for(1s); }

// two calls below are the same
auto fut1 = std::async(f);
auto fut2 = std::async(std::launch::async | std::launch::deferred, f);

// if f is deferred, fut.wait_for will always return std::future_status::deferred, so the loop will
// never terminate
auto fut = std::async(f);
while (fut.wait_for(100ms) != std::future_status::ready) {
  // ...
}

// fix the issue
auto fut = std::async(f);
if (fut.wait_for(0s) == std::future_status::deferred) {
  // ...
} else {
  while (fut.wait_for(100ms) != std::future_status::ready) {
    // task is neither deferred nor ready,
    // so do concurrent work until it's ready
  }
  // fut is ready
}

// C++11 style
template <typename F, typename... Ts>
inline std::future<typename std::result_of<F(Ts...)>::type> reallyAsync(F&& f, Ts&&... params) {
  return std::async(std::launch::async, std::forward<F>(f), std::forward<Ts>(params)...);
}

// C++14 style
template <typename F, typename... Ts>
inline auto reallyAsync(F&& f, Ts&&... params) {
  return std::async(std::launch::async, std::forward<F>(f), std::forward<Ts>(params)...);
}
```

**Item 37: Make `std::threads` unjoinable on all paths.**

- Make `std::thread`s unjoinable on all paths.

- `join`-on-destruction can lead to difficult-to-debug performance anomalies.

- detach-on-destruction can lead to difficult-to-debug undefined behavior.

- Declare `std::thread` objects last in lists of data members.

```c++
constexpr auto tenMillion = 10000000;

bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion) {
  std::vector<int> goodVals;
  std::thread t([&filter, maxVal, &goodVals] {
    for (auto i = 0; i <= maxVal; ++i) {
      if (filter(i)) {
        goodVals.push_back(i);
      }
    }
  });

  auto nh = t.native_handle();

  if (conditionsAreSatisfied()) {
    t.join();
    performComputation(goodVals);
    return true;
  }

  return false;
}

class ThreadRAII {
 public:
  enum class DtorAction { join, detach };

  ThreadRAII(std::thread&& t, DtorAction a) : action(a), t(std::move(t)) {}

  ~ThreadRAII() {
    if (t.joinable()) {
      if (action == DtorAction::join) {
        t.join();
      } else {
        t.detach();
      }
    }
  }

  ThreadRAII(ThreadRAII&&) = default;
  ThreadRAII& operator=(ThreadRAII&&) = default;

  std::thread& get() { return t; }

 private:
  DtorAction action;
  std::thread t;
};

// then use RAII class to rewrite doWork()
```

**Item 38: Be aware of varying thread handle destructor behavior.**

- Future destructors normally just destroy the future's data members.

- The final future referring to a shared state for a non-deferred task launched via `std::async` blocks until the task completes.

```c++
int calcValue();

// a std::packaged_task object prepares a function (or other callable object) for asynchronous
// execution by wrapping it such that its result is put into a shared object
std::packaged_task<int()> pt(calcValue);

auto fut = pt.get_future();  // get future for pt

std::thread t(std::move(pt));  // run pt on t

// client's decision
// the decision among termination, joining, or detaching will be made in the code that manipulates
// the std::thread on which the std::packaged_task is typically run
```

**Item 39: Consider `void` futures for one-shot event communication.**

- For simple event communication, condvar-based designs require a superfluous mutex, impose constraints on the relative progress of detecting and reacting tasks, and require reacting tasks to verify that the event has taken place.

- Designs employing a flag avoid those problems, but are based on polling, not blocking.

- A condvar and flag can be used together, but the resulting communications mechanism is somewhat stilted.

- Using `std::promise`s and futures dodges these issues, but the approach uses heap memory for shared states, and it's limited to one-shot communication.

```c++
std::promise<void> p;

void react();

void detect() {
  auto sf = p.get_future().share();

  std::vector<std::thread> vt;

  for (int i = 0; i < threadsToRun; ++i) {
    vt.emplace_back([sf] {
      sf.wait();
      react();
    });
  }

  // ThreadRAII not used, so program is terminated if code here throws!

  p.set_value();

  for (auto& t : vt) {
    t.join();
  }
}
```

**Item 40: Use `std::atomic` for concurrency, `volatile` for special memory.**

- `std::atomic` is for data accessed from multiple threads without using mutexes. It's a tool for writing concurrent software.

- `volatile` is for memory where reads and writes should not be optimized away. It's a tool for working with special memory.

## CH8: Tweaks

**Item 41: Consider pass by value for copyable parameters that are cheap to move and always copied.**

**Item 42: Consider emplacement instead of insertion.**