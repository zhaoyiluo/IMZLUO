---
title: "[NOTE] More Effective Cpp, First Edition"
subtitle: "35 New Ways to Improve Your Programs and Designs"
date: 2021-08-01T00:13:13+08:00
lastmod: 2021-08-01T00:13:13+08:00
draft: false
authors: ["zhaoyiluo"]
description: ""

tags: []
categories: ["C++"]
series: []

featuredImage: ""
featuredImagePreview: ""
---

## CH1: Basics

Item 1: Distinguish between pointers and references

- A reference must always refer to some object. The fact that there is no such thing as a null reference implies that it can be more efficient to use references than to use pointers.

- A pointer may be reassigned to refer to different objects. A reference, however, always refers to the object with which it is initialized.

- When you're implementing certain operators, there are some situations in which you should use a reference. The most common example is `operator[]`.

  ```c++
  // Undefined behavior
  char* pc = 0;
  char& rc = *pc;
  
  // No need to check if it is a null reference
  void printDouble(const double& rd) { 
    std::cout << rd << std::endl; 
  }
  
  void printDouble(const double* pd) {
    if (pd) {
      std::cout << *pd << std::endl;
    }
  }
  ```

Item 2: Prefer C++-style casts

- `static_cast` has basically the same power and meaning as the general-purpose C-style cast.

- `const_cast` is used to cast away the `const`ness or `volatile`ness of an expression.

- `dynamic_cast` is used to perform safe casts down or across an inheritance hierarchy.

  - Failed casts are indicated by a null pointer (when casting pointers) or an exception (when casting references).
  - They cannot be applied to types lacking virtual functions, nor can they cast away `const`ness.

- `reinterpret_cast` is used to perform type conversions whose result is nearly always implementation-defined.

  ```c++
  // Example: static_cast
  int first_number = 1, second_number = 1;
  double result = static_cast<double>(first_number) / second_number;
  
  // Example: const_cast
  const int val = 10;
  const int *const_ptr = &val;
  int *nonconst_ptr = const_cast<int *>(const_ptr);
  
  // Example: dynamic_cast
  class Base {
    virtual void DoIt() { std::cout << "This is Base" << std::endl; }
  };
  class Derived : public Base {
    virtual void DoIt() { std::cout << "This is Derived" << std::endl; }
  };
  Base *b = new Derived();
  Derived *d = dynamic_cast<Derived *>(b);
  
  // Example: reinterpret_cast
  int *a = new int();
  void *b = reinterpret_cast<void *>(a);  // the value of b is unspecified
  int *c = reinterpret_cast<int *>(b);    // a and c contain the same value
  ```

Item 3: Never treat arrays polymorphically

Item 4: Avoid gratuitous default constructors

## CH2: Operators

Item 5: Be wary of user-defined conversion functions

Item 6: Distinguish between prefix and postfix forms of increment and decrement operators

Item 7: Never overload `&&`, `||`, or `,`

Item 8: Understand the different meanings of `new` and `delete`

## CH3: Exceptions

Item 9: Use destructors to prevent resource leaks

Item 10: Prevent resource leaks in constructors

Item 11: Prevent exceptions from leaving destructors

Item 12: Understand how throwing an exception differs from passing a parameter or calling a virtual function

Item 13: Catch exception specifications judiciously

Item 14: Use exception specifications judiciously

Item 15: Understand the costs of exception handling

## CH4: Efficiency

Item 16: Remember the 80-20 rule

Item 17: Consider using lazy evaluation

Item 18: Amortize the cost of expected computations

Item 19: Understand the origin of temporary objects

Item 20: Facilitate the return value optimization

Item 21: Overload to avoid implicit type conversions

Item 22: Consider using `op=` instead of strand-alone `op`

Item 23: Consider alternative libraries

Item 24: Understand the costs of virtual functions, multiple inheritance, virtual base classes, and RTTI

## CH5: Techniques

Item 25: Virtualizing constructors and non-member functions

Item 26: Limiting the number of objects of a class

Item 27: Requiring or prohibiting heap-based objects

Item 28: Smart pointers

Item 29: Reference counting

Item 30: Proxy classes

Item 31: Making functions virtual with respect to more than one object

## CH6: Miscellany

Item 32: Program in the future tense

Item 33: Make non-leaf classes abstract

Item 34: Understand how to combine C++ and C in the same program

Item 35: Familiarize yourself with the language standard