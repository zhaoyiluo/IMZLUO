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

- The language specification says the result of deleting an array of derived class objects through a base class pointer is undefined.

  ```c++
  class BST {
    friend std::ostream& operator<<(std::ostream& s, const BST& data);
  };
  class BalancedBST : public BST {};
  
  // Note: array[i] is really just shorthand for an expression involving pointer
  // arithmetic, and it stands for *(array+i)
  void printBSTArray(std::ostream& s, const BST array[], int numElements) {
    for (int i = 0; i < numElements; ++i) {
      s << array[i];
    }
  }
  
  BalancedBST bBSTArray[10];
  // They'd assume each object in the array is the size of BST, but each object
  // would actually be the size of a BalancedBST
  printBSTArray(std::cout, bBSTArray, 10);
  ```

Item 4: Avoid gratuitous default constructors

- If a class lacks a default constructor, its use may be problematic in three contexts.

  - The creation of arrays.
    - There's no way to specify constructor arguments for objects in arrays.
  - Ineligible for use with many template-based container classes.
    - It's common requirement for such templates that the type used to instantiate the template provide a default constructor.
  - Virtual base classes lacking default constructors are a pain to work with.
    - The arguments for virtual base class constructors must be provided by the most derived class of the object being constructed.

  ```c++
  // Example: the creation of arrays
  class EquipmentPiece {
   public:
    EquipmentPiece(int IDNumber);
    // ...
  };
  
  // Solution 1
  // Provide the necessary arguments at the point where the array is defined
  int ID1, ID2, ID3, ..., ID10;
  // ...
  EquipmentPiece bestPieces[] = {ID1, ID2, ID3, ..., ID10};
  
  // Solution 2
  // Use an array of pointers instead of an array of objects
  typedef EquipmentPiece* PEP;
  PEP bestPieces[10];             // on the stack
  PEP* bestPieces = new PEP[10];  // on the heap
  for (int i = 0; i < 10; ++i) {
    bestPieces[i] = new EquipmentPiece(/* ID Number */);
  }
  
  // Solution 3
  // Allocate the raw memory for the array, then use "placement new" to construct
  void* rawMemory = operator new[](10 * sizeof(EquipmentPiece));
  EquipmentPiece* bestPieces = static_cast<EquipmentPiece*>(rawMemory);
  for (int i = 0; i < 10; ++i) {
    new (&bestPieces[i]) EquipmentPiece(/* ID Number */);
  }
  for (int i = 9; i >= 0; --i) {
    bestPieces[i].~EquipmentPiece();
  }
  operator delete[] bestPieces;
  ```

## CH2: Operators

Item 5: Be wary of user-defined conversion functions

- Two kinds of functions allow compilers to perform implicit type conversions.

  - A single-argument constructor is a constructor that may be called with only one argument.
  - An implicit type conversion operator is simply a member function with a strange-looking name: the word `operator` followed by a type specification.

- Constructors can be declared `explicit`, and if they are, compilers are prohibited from invoking them for purposes of implicit type conversion.

- Proxy objects can give you control over aspects f your software's behavior, in this case implicit type conversions, that is otherwise beyond our grasp.

  - No sequence of conversions is allowed to contain more than one user-defined conversion.

  ```c++
  // The single-argument constructors
  class Name {
   public:
    Name(const std::string& s);
    // ...
  };
  
  // The implicit type conversion operators
  class Rational {
   public:
    operator double() const;
  };
  
  // Usage of `explicit`
  template <class T>
  class Array {
   public:
    // ...
    explicit Array(int size);
    // ...
  };
  
  // Usage of proxy classes
  template <class T>
  class Array {
   public:
    class ArraySize {  // this class is new
     public:
      ArraySize(int numElements) : theSize(numElements) {}
      int size() const { return theSize; }
  
     private:
      int theSize;
    };
  
    Array(int lowBound, int highBound);
    Array(ArraySize size);  // note new declaration
  };
  ```

Item 6: Distinguish between prefix and postfix forms of increment and decrement operators

- The prefix forms return a reference, while the post forms return a `const` object.

- While dealing with user-defined types, prefix increment should be used whenever possible, because it's inherently more efficient.

- Postfix increment and decrement should be implemented in terms of their prefix counterparts.

  ```c++
  class UPInt {
   public:
    UPInt& operator++();          // prefix ++
    const UPInt operator++(int);  // postfix ++
    UPInt& operator--();          // prefix --
    const UPInt operator--(int);  // postfix --
    UPInt& operator+=(int);       // a += operator for UPInts and ints
  };
  
  UPInt& UPInt::operator++() {
    *this += 1;
    return *this;
  }
  
  const UPInt UPInt::operator++(int) {
    UPInt oldValue = *this;
    ++(*this);
    return oldValue;
  }
  ```

Item 7: Never overload `&&`, `||`, or `,`

- C++ employs short-circuit evaluation of boolean expressions, but function call semantics differ from short-circuit semantics in two crucial ways.
  - When a function call is made, all parameters must be evaluated, so when calling the function `operators&&` and `operator||`, both parameters are evaluated.
  - The language specification leaves undefined the order of evaluation of parameters to a function call, so there is no way of knowing whether `expression1` or `expression2` well be evaluated first. 
- An expression containing a comma is evaluated by first evaluating the part of the expression to the left of the comma, then evaluating the expression to the right of the comma; the result of the overall comma expression is the value of the expression on the right.

Item 8: Understand the different meanings of `new` and `delete`

- The `new` you are using is the `new` operator.

  - First, it allocates enough memory to hold an object of the type requested. The name of the function the `new` operator calls to allocate memory is `operator new`.
  - Second, it calls a constructor to initialize an object in the memory that was allocated.
  - A special version of `operator new` called placement `new` allows you to call a constructor directly.

- The function `operator delete` is to the built-in `delete` operator as `operator new` is to the `new` operator.

- There is only one global `operator new`, so if you decide to claim it as your own, you instantly render your software incompatible with any library that makes the same decision.

  ```c++
  // `new` operator:
  // Step 1: void *memory = operator new(sizeof(std::string))
  // Step 2: call std::string::string("Memory Management") on *memory
  // Step 3: std::string *ps = static_cast<std::string*>(memory)
  //
  // `delete` operator:
  // Step 1: ps->~string()
  // Step 2: operator delete(ps)
  
  // Placement `new`:
  class Widget {
   public:
    Widget(int widgetSize);
    // ...
  };
  // It's just a use of the `new` operator with an additional argument
  // (buffer) is being specified for the implicit call that the `new`
  // operator makes to `operator new`
  Widget* constructWidgetInBuffer(void* buffer, int widgetSize) {
    return new (buffer) Widget(widgetSize);
  }
  ```

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