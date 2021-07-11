---
title: "<Book Notes> Effective C++, Third Edition"
subtitle: "55 Specific Ways to Improve Your Programs and Designs"
date: 2021-04-19T00:03:54+08:00
lastmod: 2021-04-19T00:03:54+08:00
draft: false
authors: ["zhaoyiluo"]
description: ""

tags: []
categories: ["C++"]
series: []

featuredImage: ""
featuredImagePreview: ""
---

## Chapter 1: Accustoming Yourself to C++

**Item 1: View C++ as a federation of languages.**

- C++ was just C with some object-oriented features tacked on.
- View C++ as a federation of related languages: C, Object-Oriented C++, Template C++, and The STL.
- Rules for effective C++ programming vary, depending on the part of C++ you are using.

**Item 2: Prefer `const`s, `enum`s, and `inline`s to `#define`s.**

- Prefer the compiler to the preprocessor as `#define` may be treated as if it's not part of the language *per se*.
  - The symbolic name may be removed by the preprocessor before the source code ever gets to a compiler.
  - Not yet time to retire the preprocessor but should definitely give it long and frequent vacations.
- Prefer `const` objects or `enum`s to `#define`s for simple constants.
- Prefer `inline` functions to `#define`s for function-like macros.
- Use of the constant may yield smaller code than using a `#define`.
  - Define constant pointers and prefer `string` objects to their `char*`-based progenitors.
  - Make a class-specific constant a member and make it a `static` member to ensure there's at most one copy.
- *The `enum` hack* takes advantage of the fact that the values of an enumerated type can be used where `int`s are expected.
  - Behave in some ways more like a `#define` than a `const` does, e.g., illegal to take the address of an `enum`.
  - Be pragmatic, e.g., a fundamental technique of template metaprogramming.

**Item 3: Use `const` whenever possible.**

- `const` can be applied to objects at any scope, to function parameters and return types, and to member functions as a while.
- Declaring something `const` allows specifying a semantic constraint and helps compilers detect usage errors.
- Declaring an iterator `const` is like declaring a pointer `const`, e.g., declaring a `T* const` pointer.
- Declaring a parameter or local object `const` saves from annoying errors.
  - e.g., "I meant to type '`==`' but I accidently typed '`=`'" mistake.
- There are two prevailing notions regarding `const` member functions: *bitwise constness* (a.k.a. *physical constness*) and *logical constness*.
  - *bitwise constness* believes that if and only if it doesn't modify any of the object's non-`static` data members.
  - *logical constness* believes that it might modify some of the bits in the object on which it's invoked, but only in ways that clients cannot detect.
  - `mutable` frees non-`static` data members from the constraints of *bitwise constness*.
  - Compilers enforce *bitwise constness*, but you should program using *logical constness*.
- Code duplication can be avoided by having the non-`const` version call the `const` version.
  - Worth knowing the technique of implementing a non-`const` member function in terms of its `const` twin.
  - e.g., have the non-`const` version of `operator[]` call the `const` one (`static_cast`) and that brings to casting away constness (`const_cast`).

**Item 4: Make sure that objects are initialized before they're used.**

- Manually initialize objects of built-in type, because C++ only sometimes initializes them itself.
- In a constructor, prefer use of the member initialization list to assignment inside the body of the constructor.
  - Data members of an object are initialized before the body of a constructor is entered.
  - Data members that are `const` or are references must be initialized and can't be assigned.
  - List data members in the initialization list in the same order they're declared in the class.
- The relative order of initialization of non local static objects defined in different translation units is undefined.
  - A translation unit is the source code giving rise to a single object file. It's basically a single source file, plus all of its `#include` files.
- Avoid initialization order problems across translation units by replacing non-local `static` objects with local `static` objects.
  - Move each non-local `static` object into its own function, where it's declared `static`.
  - Local `static` objects are initialized when the object's definition is first encountered during a call to that function.
- The reference-returning functions are excellent candidates for inlining, especially if they're called frequently.
  - They are problematic in multithreaded systems. Solved by manually invoking all them during the single-threaded startup portion of the program.

## Chapter 2: Constructors, Destructors, and Assignment Operators

**Item 5: Know what functions C++ silently writes and calls.**

- Compilers may implicitly generate a class's default constructor, copy constructor, copy assignment operator, and destructor.
  - All these functions are both `public` and `inline`.
  - All these functions are generated only if they are needed.
- If the copy assignment in a class containing a reference member should be supported, the function should be defined manually.
  - Compilers behave similarly for classes containing `const` members.
- Compilers reject implicit copy assignment operators in derived classes that inherit from base classes declaring the copy assignment operator private.

**Item 6: Explicitly disallow the use of compiler-generated functions you do not want.**

- To disallow functionality automatically provided by compilers, declare the corresponding member functions `private` and give no implementations.
  - If the function is called inadvertently in a member or a friend function, the linker will complain.
- To move the link-time error up to compile time,  let the class inherit from `Uncopyable` class.
  - Inheritance from `Uncopyable` needn't be public.
  - `Uncopyable`'s destructor needn't be virtual. 

**Item 7: Declare destructors virtual in polymorphic base classes.**

- The purpose of virtual functions is to allow customization of derived class implementations.
- Polymorphic base classes should declare virtual destructors. If a class has any virtual functions, it should have a virtual destructor.
  - When a derived class object is deleted through a pointer to a base class with a non-virtual destructor, results are undefined.
- Classes not designed to be base classes or not designed to be used polymorphically should not declare virtual destructors.

**Item 8: Prevent exceptions from leaving destructors.**

- Destructors should never emit exceptions.
- If functions called in a destructor may throw, a reasonable idea would be to create a resource-managing class.
  - Terminate the program, typically by calling abort as it may forestall undefined behavior.
  - Swallow the exception, a bad idea as it may suppress important information.
- A better solution is to design the interface within the resource-managing class so that its clients have an opportunity to react to problems that may arise.
  - If there may be a need to handle that exception, the exception has to come from some non-destructor function.

**Item 9: Never call virtual functions during construction or destruction.**

- Don't call virtual functions during construction or destruction, because such calls will never go to a more derived class than that of the currently executing constructor or destructor.
- Compensate by having derived classes pass necessary construction information up to base class constructors.

**Item 10: Have assignment operators return a reference to *this.**

- Have assignment operators return a reference to `*this`.
  - The convention is followed by all the built-in types as well as by all the types in the standard library.

**Item 11: Handle assignment to self in operator=.**

- Make sure `operator=` is well-behaved when an object is assigned to itself.
  - Solution 1: compare addresses of source and target objects.
  - Solution 2: order statements carefully.
  - Solution 3: apply "copy and swap`" technique.
- Make sure that any function operating on more than one object behaves correctly if two or more of the objects are the same.

**Item 12: Copy all parts of an object.**

- Copying functions should be sure to copy all of an object's data members and all of its base class parts.
- Don't try to implement one of the copying functions in terms of the other. Instead, put common functionality in a third function that both call.

## Chapter 3: Resource Management

**Item 13: Use objects to manage resources.**

**Item 14: Think carefully about copying behavior in resource-managing classes.**

**Item 15: Provide access to raw resources in resource-managing classes.**

**Item 16: Use the same form in corresponding uses of new and delete.**

**Item 17: Store newed objects in smart pointers in standalone statements.**

## Chapter 4: Designs and Declarations

**Item 18: Make interfaces easy to use correctly and hard to use incorrectly.**

**Item 19: Treat class design as type design.**

**Item 20: Prefer pass-by-reference-to-const to pass-by-value.**

**Item 21: Don't try to return a reference when you must return an object.**

**Item 22: Declare data members private.**

**Item 23: Prefer non-member non-friend functions to member functions.**

**Item 24: Declare non-member functions when type conversions should apply to all parameters.**

**Item 25: Consider support for a non-throwing swap.**

## Chapter 5: Implementations

**Item 26: Postpone variable definitions as long as possible.**

**Item 27: Minimize casting.**

**Item 28: Avoid returning "handles" to object internals.**

**Item 29: Strive for exception-safe code.**

**Item 30: Understand the ins and outs of inlining.**

**Item 31: Minimize compilation dependencies between files.**

## Chapter 6: Inheritance and Object-Oriented Design

**Item 32: Make sure public inheritance models "is-a."**

**Item 33: Avoid hiding inherited names.**

**Item 34: Differentiate between inheritance of interface and inheritance of implementation.**

**Item 35: Consider alternatives to virtual functions.**

**Item 36: Never redefine an inherited non-virtual function.**

**Item 37: Never redefine a function's inherited default parameter value.**

**Item 38: Model "has-a" or "is-implemented-in-terms-of" through composition.**

**Item 39: Use private inheritance judiciously.**

**Item 40: Use multiple inheritance judiciously.**

## Chapter 7: Templates and Generic Programming

**Item 41: Understand implicit interfaces and compile-time polymorphism.**

**Item 42: Understand the two meanings of typename.**

**Item 43: Know how to access names in templatized base classes.**

**Item 44: Factor parameter-independent code out of templates.**

**Item 45: Use member function templates to accept "all compatible types."**

**Item 46: Define non-member functions inside templates when type conversions are desired.**

**Item 47: Use traits classes for information about types.**

**Item 48: Be aware of template metaprogramming.**

## Chapter 8: Customizing new and delete

**Item 49: Understand the behavior of the new-handler.**

**Item 50: Understand when it makes sense to replace new and delete.**

**Item 51: Adhere to convention when writing new and delete.**

**Item 52: Write placement delete if you write placement new.**

## Chapter 9: Miscellany

**Item 53: Pay attention to compiler warnings.**

**Item 54: Familiarize yourself with the standard library, including TR1.**

**Item 55: Familiarize yourself with Boost.**
