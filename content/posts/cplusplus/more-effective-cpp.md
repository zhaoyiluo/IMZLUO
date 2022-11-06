---
title: "[NOTE] More Effective C++"
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

**Item 1: Distinguish between pointers and references**

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

**Item 2: Prefer C++-style casts**

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

**Item 3: Never treat arrays polymorphically**

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

**Item 4: Avoid gratuitous default constructors**

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

**Item 5: Be wary of user-defined conversion functions**

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

**Item 6: Distinguish between prefix and postfix forms of increment and decrement operators**

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

**Item 7: Never overload `&&`, `||`, or `,`**

- C++ employs short-circuit evaluation of boolean expressions, but function call semantics differ from short-circuit semantics in two crucial ways.
  - When a function call is made, all parameters must be evaluated, so when calling the function `operators&&` and `operator||`, both parameters are evaluated.
  - The language specification leaves undefined the order of evaluation of parameters to a function call, so there is no way of knowing whether `expression1` or `expression2` well be evaluated first. 
- An expression containing a comma is evaluated by first evaluating the part of the expression to the left of the comma, then evaluating the expression to the right of the comma; the result of the overall comma expression is the value of the expression on the right.

**Item 8: Understand the different meanings of `new` and `delete`**

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

**Item 9: Use destructors to prevent resource leaks**

- By adhering to the rule that resources should be encapsulated inside objects, you can usually avoid resource leaks in the presence of exceptions.

**Item 10: Prevent resource leaks in constructors**

- C++ destroys only fully constructed objects, and an object isn't fully constructed until its constructor has return to completion.

- If you replace pointer class members with their corresponding `auto_ptr` objects, you fortify your constructors against resource leaks in the presence of exceptions, you eliminate the need to manually deallocate resources in destructors, and you allow `const` member pointers to be handled in the same graceful fashion as non-`const` pointers.

  ```c++
  class BookEntry;
  
  // If BookEntry's constructor throws an exception, pb will be the null pointer,
  // so deleting it in the catch block does nothing except make you feel better
  // about yourself
  void testBookEntryClass() {
    BookEntry* pb = 0;
    try {
      pb = new BookEntry(/* ... */);
      // ...
    } catch (/* ... */) {
      delete pb;
      throw;
    }
    delete pb;
  }
  ```

**Item 11: Prevent exceptions from leaving destructors**

- You must write your destructors under the conservative assumption that an exception is active, because if an exception is thrown while another is active, C++ calls the `terminate` function.

- If an exception is throw from a destructor and is not caught there, that destructor won't run to completion.

- There are two good reasons for keeping exceptions from propagating out of destructors.

  - It prevents `terminate` from being called during the stack-unwinding part of exception propagation.
  - It helps ensure that desturctors always accomplish everything they are supposed to accomplish.

  ```c++
  class MyClass {
   public:
    void doSomeWork();
    // ...
  };
  
  try {
    MyClass obj;
    // If an exception is thrown from doSomeWork, before execution moves out from the try block,
    // the destructor of obj needs to be called as obj is a properly constructed object
    // What if an exception is also thrown from the destructor of obj?
    obj.doSomeWork();
    // ...
  } catch (std::exception& e) {
    // do error handling
  }
  ```

**Item 12: Understand how throwing an exception differs from passing a parameter or calling a virtual function**

- Exception objects are always copied; when caught by value, they are copied twice. Objects passed to function parameters need not be copied at all.

  - When an object is copied for use as an exception, the copying is performed by the object's copy constructor. This copy constructor is the one in the class corresponding to the object's static type, not its dynamic type.
  - Passing a temporary object to a non-`const` reference parameter is not allowed for function calls, but it is for exceptions.

- Objects thrown as exceptions are subject to fewer forms of type conversion than are objects passed to functions.

  - A `catch` clause for base class exceptions is allowed to handle exceptions of derived class types.
  - From a typed to an untyped pointer.

- `catch` clauses are examined in the order in which they appear in the source ode, and the first one that can succeed is selected for execution.

  - Never put a `catch` clause for a base class before a `catch` clause for a derived class.

  ```c++
  // Difference 1
  class Widget {};
  class SpecialWidget : public Widget {};
  
  void passAndThrowWidget() {
    SpecialWidget localSpecialWidget;
    // ...
    Widget& rw = localSpecialWidget;  // rw refers to a SpecialWidget
    throw rw;                         // this throws an exception of type Widget
  }
  
  try {
    passAndThrowWidget();
  }
  catch (Widget& w) {    // catch Widget exceptions
    // ...               // handle the exception
    throw;               // rethrow the exception so it continues to propagate
  } catch (Widget& w) {  // catch Widget exceptions
    // ...               // handle the exception
    throw w;             // propagate a copy of the caught exception
  }
  
  // Difference 2
  // Can catch errors of type runtime_error, range_error, or overflow_error
  catch (std::runtime_error);
  catch (std::runtime_error&);
  catch (const std::runtime_error&);
  // Can catch any exception that's a pointer
  catch(const void*);
  
  // Difference 3
  try {
    // ...
  } catch (std::invalid_argument& ex) {
    // ...
  } catch (std::logic_error& ex) {
    // ...
  }
  ```

**Item 13: Catch exception specifications judiciously**

- If you try to catch exceptions by pointer, you must define exception objects in a way that guarantees the objects exist after control leaves the functions throwing pointers to them. Global and static objects work fine, but it's easy for you to forget the constraint.
  
  - The four standard exceptions, `bad_alloc`, `bad_cast`, `bad_typeid`, and `bad_exception` are all objects, not pointers to objects.
  
- If you try to catch exceptions by value, it requires that exception objects be copied twice each time they're thrown and also gives rise to the specter of the *slicing problem*.

- If you try to catch exceptions by reference, you sidestep questions about object deletion that leave you demand if you do and damned if you don't; you avoid slicing exception objects; you retain the ability to catch standard exceptions; and you limit the number of times exception objects need to be copied.

**Item 14: Use exception specifications judiciously**

- The default behavior for `unexpected` is to call `terminate`, and the default behavior for `terminate` is to call `abort`, so the default behavior for a program with a violated exception specification is to halt.

- Compilers only partially check exception usage for consistency with exception specifications. What they do not check for is a call to a function that might violate the exception specification of the function making the call.

- Avoid putting exception specifications on templates that take type arguments.

- Omit exception specifications on functions making calls to functions that themselves lack exception specifications.

  - One case that is easy to forget is when allowing users to register callback functions.

- Handle exceptions "the system" may throw.

- If preventing exceptions isn't practical, you can exploit the fact that C++ allows you to replace unexpected exceptions with exceptions of a different type.

- Exception specifications result in `unexpected` being invoked even when a higher-level caller is prepared to cope with the exception that's arisen.

  ```c++
  // Partially check
  extern void f1();
  void f2() throw(int) {
    // ...
    f1();  // legal even though f1 might throw something besides an int
    // ...
  }
  
  // A poorly designed template wrt exception specifications
  // Overloaded operator& may throw an exception
  template <class T>
  bool operator==(const T& lhs, const T& rhs) throw() {
    return &lhs == &rhs;
  }
  
  // Replace the default unexpected function
  class UnexpectedException {
  };  // all unexpected exception objects will be replaced by objects of this type
  void convertUnexpected() {
    throw UnexpectedException();
  }  // function to call if an unexpected exception is thrown
  std::set_unexpected(convertUnexpected);
  
  // Suppose some function called by logDestruction throws an exception
  // When this unanticipated exception propagates through logDestruction, unexpected
  // will be called
  class Session {
   public:
    ~Session();
    // ...
   private:
    static void logDestruction(Session* objAddr) throw();
  };
  
  Session::~Session() {
    try {
      logDestruction(this);
    } catch (/* ... */) {
      // ...
    }
  }
  ```

**Item 15: Understand the costs of exception handling**

- To minimize your exception-related costs, compile without support for exceptions when that is feasible; limit your use of `try` blocks and exception specifications to those locations where you honestly need them; and throw exceptions only under conditions that are truly exceptional.

## CH4: Efficiency

**Item 16: Remember the 80-20 rule**

- The overall performance of your software is almost always determined by a small part of its constituent code.

- The best way to guard against these kinds of pathological results is to profile your software using as many data sets as possible.

**Item 17: Consider using lazy evaluation**

- When you employ lazy evaluation, you write your classes in such a way that they defer computations until the results of those computations are required.

  - To avoid unnecessary copying of objects.
  - To distinguish reads from writes using `operator[]`.
  - To avoid unnecessary reads from databases.
  - To avoid unnecessary numerical computations.

- Lazy evaluation is only useful when there's a reasonable chance your software will be asked to perform computations that can be avoided.

  ```c++
  // Example 1: Reference Counting
  class String;
  String s1 = "Hello";
  String s2 = s1;           // call String copy constructor
  std::cout << s1;          // read s1's value
  std::cout << s1 + s2;     // read s1's and s2's values
  s2.convertToUpperCase();  // don't bother to make a copy of something until you really need one
  
  // Example 2: Distinguish Reads from Writes
  String s = "Homer's Iliad";
  std::cout << s[3];  // call operator[] to read s[3]
  s[3] = 'x';         // call operator[] to write s[3]
  
  // Example 3: Lazy Fetching
  class ObjectID;
  class LargeObject {
   public:
    LargeObject(ObjectID id);
    const std::string& field1() const;
    int field2() const;
    double field3() const;
    const std::string& field4() const;
    // ...
   private:
    ObjectID oid;
    mutable std::string* field1Value;
    mutable int* field2Value;
    mutable double* field3Value;
    mutable std::string* field4Value;
  };
  
  LargeObject::LargeObject(ObjectID id)
      : oid(id), field1Value(0), field2Value(0), field3Value(0), field4Value(0) {}
  
  const std::string& LargeObject::field1() const {
    if (field1Value == 0) {
      // Read the data for field 1 from the database and make field1Value point to it
    }
    return *field1Value;
  }
  
  // Example 4: Lazy Expression Evaluation
  template <class T>
  class Matrix {};
  
  Matrix<int> m1(1000, 1000);
  Matrix<int> m2(1000, 1000);
  
  Matrix<int> m3 = m1 + m2;  // set up a data structure inside m3 that includes some information
  
  Matrix<int> m4(1000, 1000);
  m3 = m4 * m1;  // no need to actually calculate the result of m1 + m2 previously
  ```

**Item 18: Amortize the cost of expected computations**

- Over-eager evaluation is a technique for improving the efficiency of programs when you must support operations whose results are almost always needed or whose results are often needed more than once.

  - Caching values that have already been computed and are likely to be needed again.
  - Prefetching demands a place to put the things that are prefetched, but it reduces the time need to access those things.

**Item 19: Understand the origin of temporary objects**

- True temporary objects in C++ are invisible -- they don't appear in your source code. They arise whenever a non-heap object is created but no named.

  - When implicit type conversions are applied to make function calls succeed.

  - When functions return objects.

    ```c++
    // Situation 1:
    unsigned int countChar(const std::string& str, char ch);
    char buffer[MAX_STRING_LEN];
    char c;
    
    // Create a temporary object of type string and the object is initialized by calling
    // the string constructor with buffer as its argument
    // These conversions occur only when passing objects by value or when passing to a
    // reference-to-const parameter
    countChar(buffer, c);
    
    // Situation 2:
    class Number;
    // The return value of this function is a temporary
    const Number operator+(const Number& lhs, const Number& rhs);
    ```

**Item 20: Facilitate the return value optimization**

- It is frequently possible to write functions that return objects in such a way that compilers can eliminate the cost of the temporaries. The trick is to return constructor arguments instead of objects.

  ```c++
  class Rational {
   public:
    Rational(int numerator = 0, int denominator = 1);
    // ...
    int numerator() const;
    int denominator() const;
  
   private:
    int numerator;
    int denominator;
  };
  
  // Usage 1: without RVO
  // Step 1: call constructor to initialize result
  // Step 2: call copy constructor to copy local variable result from callee to caller (temporary
  // variable)
  // Step 3: call destructor to destruct local variable result
  const Rational operator*(const Rational& lhs, const Rational& rhs) {
    Rational result(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
  
    return result;
  }
  
  // Usage 2: without RVO
  // Step: caller directly initialize variable defined by the return expression inside the allocated
  // memory
  const Rational operator*(const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
  }
  ```

**Item 21: Overload to avoid implicit type conversions**

- By declaring several functions, each with a different set of parameter types to eliminate the need for type conversions.

  ```c++
  class UPInt {
   public:
    UPInt();
    UPInt(int value);
    // ...
  };
  
  // Overloaded functions to eliminate type conversions
  const UPInt operator+(const UPInt& lhs, const UPInt& rhs);
  const UPInt operator+(const UPInt& lhs, int rhs);
  const UPInt operator+(int lhs, const UPInt& rhs);
  
  // Not allowed
  // Every overloaded operator must take at least one argument of a user-defined type
  const UPInt operator+(int lhs, int rhs);
  ```

**Item 22: Consider using `op=` instead of strand-alone `op`**

- A good way to ensure that the natural relationship between the assignment version of an operator (e.g., `operator+=`) and the stand-alone version (e.g., `operator+`) exists is to implement the latter in terms of the former.

- In general, assignment versions of operators are more efficient than stand-alone versions, because stand-alone versions must typically return a new object, and that costs us the construction and destruction of a temporary. Assignment versions of operators write to their left-hand argument, so there is no need to generate a temporary to hold the operator's return value.

- By offering assignment versions of operators as well as stand-alone versions, you allow clients of your classes to make the difficult trade-off between efficiency and convenience.

  ```c++
  class Rational {
   public:
    // ...
    Rational& operator+=(const Rational& rhs);
    Rational& operator-=(const Rational& rhs);
  };
  
  const Rational operator+(const Rational& lhs, const Rational& rhs) { return Rational(lhs) += rhs; }
  const Rational operator-(const Rational& lhs, const Rational& rhs) { return Rational(lhs) += rhs; }
  
  // Eliminate the need to write the stand-alone functions
  // The corresponding stand-alone operator will automatically be generated if it's needed
  template <class T>
  const T operator+(const T& lhs, const T& rhs) {
    return T(lhs) += rhs;
  }
  template <class T>
  const T operator-(const T& lhs, const T& rhs) {
    return T(lhs) -= rhs;
  }
  ```

**Item 23: Consider alternative libraries**

- Different libraries embody different design decisions regarding efficiency, extensibility, portability, type safety, and other issues. You can sometimes significantly improve the efficiency of your software by switching to libraries whose designers gave more weight to performance considerations than to other factors.

**Item 24: Understand the costs of virtual functions, multiple inheritance, virtual base classes, and RTTI**

| Feature              | Increases <br />Size of Objects | Increases<br />Per-Class Data | Reduces<br />Inlining |
| -------------------- | ------------------------------- | ----------------------------- | --------------------- |
| Virtual Functions    | Yes                             | Yes                           | Yes                   |
| Multiple Inheritance | Yes                             | Yes                           | No                    |
| Virtual Base Classes | Yes                             | No                            | No                    |
| RTTI                 | No                              | Yes                           | No                    |

## CH5: Techniques

**Item 25: Virtualizing constructors and non-member functions**

- A virtual constructor is a function that creates different types of objects depending on the input it is given.

- A virtual copy constructor returns a pointer to a new copy of the object invoking the function.

- No longer must a derived class's redefinition of a base class's virtual function declare the same return type.

- You write virtual functions to do the work, then write a non-virtual function that does nothing but call the virtual function.

  ```c++
  class NLComponent {
   public:
    virtual NLComponent* clone() const = 0;  // virtual copy constructor
    virtual std::ostream& print(std::ostream& s) const = 0;
    // ...
  };
  
  class TextBlock : public NLComponent {
   public:
    virtual TextBlock* clone() const { return new TextBlock(*this); }  // virtual copy constructor
    virtual std::ostream& print(std::ostream& s) const;
    // ...
  };
  
  class Graphic : public NLComponent {
   public:
    virtual Graphic* clone() const { return new Graphic(*this); }  // virtual copy constructor
    virtual std::ostream& print(std::ostream& s) const;
    // ...
  };
  
  class NewsLetter {
   public:
    NewsLetter(std::istream& str);
  
   private:
    static std::list<NLComponent*> components;
  };
  
  NLComponent* readComponent(std::istream& str);  // virtual constructor
  
  NewsLetter::NewsLetter(std::istream& str) {
    while (str) {
      components.push_back(readComponent(str));
    }
  }
  
  // Make non-member functions act virtual
  inline std::ostream& operator<<(std::ostream& s, const NLComponent& c) { return c.print(s); }
  ```

**Item 26: Limiting the number of objects of a class**

- The easiest way to prevent objects of a particular class from being created is to declare the constructors of that class private.

  - If you create an inline non-member function containing a local static object, you may end up with more than one copy of the static object in your program due to internal linkage.

- The object construction can exist in three different contexts: on their own, as base class parts of more derived objects, and embedded inside larger objects.

  ```C++
  // Allowing zero or one objects
  class PrintJob {
   public:
    PrintJob(const std::string& whatToPrint);
    // ...
  };
  
  class Printer {
   public:
    void submitJob(const PrintJob& job);
    void reset();
    void performSelfTest();
    // ...
  
    // Design choice 1: be friend
    friend Printer& thePrinter();
    // Design choice 2: be static
    static Printer& thePrinter();
  
   private:
    Printer();
    Printer(const Printer& rhs);
    // ...
  };
  
  // It's important that the single Printer object be static in a function and not in a class.
  // An object that's static in a class is, for all intents and purposes, always constructed (and
  // destructed), even if it's never used. In contrast, an object that's static in a function is
  // created the first time through the function, so if the function is never called, the object is
  // never created
  
  // Design choice 1: be friend
  Printer& thePrinter() {
    static Printer p;
    return p;
  }
  thePrinter().reset();
  thePrinter().submitJob(std::string());
  
  // Design choice 2: be static
  Printer& Printer::thePrinter() {
    static Printer p;
    return p;
  }
  Printer::thePrinter().reset();
  Printer::thePrinter().submitJob(std::string());
  ```

  ```c++
  // Allowing objects to come and go
  class Printer {
   public:
    class TooManyObjects {};
    static Printer* makePrinter();
    static Printer* makePrinter(const Printer& rhs);
    // ...
  
   private:
    static unsigned int numObjects;
    static const int maxObjects = 10;
  
    Printer();
    Printer(const Printer& rhs);
  };
  
  unsigned int Printer::numObjects = 0;
  const int Printer::maxObjects;
  
  Printer::Printer() {
    if (numObjects >= maxObjects) {
      throw TooManyObjects();
    }
    // ...
  }
  
  Printer::Printer(const Printer& rhs) {
    if (numObjects >= maxObjects) {
      throw TooManyObjects();
    }
    // ...
  }
  
  Printer* Printer::makePrinter() { return new Printer; }
  
  Printer* Printer::makePrinter(const Printer& rhs) { return new Printer(rhs); }
  ```

  ```c++
  // An object-counting base class
  template <class BeingCounted>
  class Counted {
   public:
    class TooManyObjects {};
    static int objectCount() { return numObjects; }
  
   protected:
    Counted();
    Counted(const Counted& rhs);
    ~Counted() { --numObjects; }
  
   private:
    static int numObjects;
    static int maxObjects;
    void init();
  };
  
  template <class BeingCounted>
  Counted<BeingCounted>::Counted() {
    init();
  }
  
  template <class BeingCounted>
  Counted<BeingCounted>::Counted(const Counted<BeingCounted>&) {
    init();
  }
  
  template <class BeingCounted>
  void Counted<BeingCounted>::init() {
    if (numObjects >= maxObjects) {
      throw TooManyObjects();
    }
    ++numObjects;
  }
  
  class PrintJob;
  
  class Printer : private Counted<Printer> {
   public:
    static Printer* makePrinter();
    static Printer* makePrinter(const Printer& rhs);
    ~Printer();
    void submitJob(const PrintJob& job);
    void reset();
    void performSelfTest();
  
    // Make public for clients of Printer
    using Counted<Printer>::objectCount;
    using Counted<Printer>::TooManyObjects;
  
   private:
    Printer();
    Printer(const Printer& rhs);
  };
  ```

**Item 27: Requiring or prohibiting heap-based objects**

- Restricting access to a class's destructor or its constructors also prevents both inheritance and containment.

  - The inheritance problem can be solved by making a class's destructor protected.
  - The classes that need to contain heap-based objects can be modified to contain pointers to them.

- There's not only no portable way to determine whether an object is on the heap, there isn't even a semi-portable way that works most of the time.

  - A mixin ("mix in") class offers derived classes the ability to determine whether a pointer was allocated from `operator new`.

- The fact that the stack-based class's `operator new` is private has no effect on attempts to allocate objects containing them as members.

  ```c++
  // Require heap-based objects
  class UPNumber {
   public:
    UPNumber();
    UPNumber(int initValue);
    UPNumber(double initValue);
    UPNumber(const UPNumber& rhs);
    // pseudo-destructor
    void destroy() const { delete this; }
    // ...
  
   protected:
    ~UPNumber();
  };
  
  // Derived classes have access to protected members
  class NonNegativeNumber : public UPNumber {};
  
  // Contain pointers to head-based objects
  class Asset {
   public:
    Asset(int initValue);
    ~Asset();
    // ...
  
   private:
    UPNumber* value;
  };
  
  Asset::Asset(int initValue) : value(new UPNumber(initValue)) {}
  Asset::~Asset() { value->destroy(); }
  ```

  ```c++
  // Determine whether an object is on the heap
  class HeapTracked {
   public:
    class MissingAddress {};
    virtual ~HeapTracked() = 0;
    static void *operator new(size_t size);
    static void operator delete(void *ptr);
    bool isOnHeap() const;
  
   private:
    typedef const void *RawAddress;
    static std::list<RawAddress> addresses;
  };
  
  std::list<HeapTracked::RawAddress> HeapTracked::addresses;
  
  HeapTracked::~HeapTracked() {}
  
  void *HeapTracked::operator new(size_t size) {
    void *memPtr = ::operator new(size);
    addresses.push_back(memPtr);
    return memPtr;
  }
  
  void HeapTracked::operator delete(void *ptr) {
    std::list<RawAddress>::iterator it = std::find(addresses.begin(), addresses.end(), ptr);
    if (it != addresses.end()) {
      addresses.erase(it);
      ::operator delete(ptr);
    } else {
      throw MissingAddress();
    }
  }
  
  // dynamic_cast is applicable only to pointers to objects that have at least one virtual function
  // and it gives us a pointer to the beginning of the memory for the current object
  bool HeapTracked::isOnHeap() const {
    const void *rawAddress = dynamic_cast<const void *>(this);
    std::list<RawAddress>::iterator it = std::find(addresses.begin(), addresses.end(), rawAddress);
    return it != addresses.end();
  }
  
  class Asset : public HeapTracked {
   private:
    UPNumber value;
    // ...
  };
  
  void inventoryAsset(const Asset *ap) {
    if (ap->isOnHeap()) {
      // ...
    } else {
      // ...
    }
  }
  ```

  ```c++
  // Prohibit heap-based objects
  class Asset {
   public:
    Asset(int initValue);
    // ...
  
   private:
    UPNumber value;  // has private operator new
  };
  
  Asset* pa = new Asset(100);  // calls Asset::operator new or ::operator new,
                               // not UPNumber::operator new
  ```

**Item 28: Smart pointers**

- Passing `auto_ptr`s by value, then, is something to be done only if you're sure you want to transfer ownership of an object to a (transient) function parameter.

- The `operator*` function just returns a reference to the pointed-to object such that `pointee` need not point to an object of type `T`, while it may point to an object of a class derived from `T`. The `operator->` function returns a dumb pointer to an object or another smart pointer object as it must be legal to apply the member-selection operator (->) to it.

- Overload `operator!` for your smart pointer classes so that `operator!` returns `true` if and only if the smart pointer on which it's invoked is null.

- Do not provide implicit conversion operators to dumb pointers unless there is a compelling reason to do so.

- Use member function templates to generate smart pointer conversion functions for inheritance-based type conversions.

  - Smart pointers employ member functions as conversion operators, and as far as C++ compilers are concerned, all calls to conversion functions are equally good. The best we can do is to use member templates to generate conversion functions, then use casts in those cases where ambiguity results.

- Implement smart pointers by having each smart pointer-to-`T`-class publicly inherit from a corresponding smart pointer-to-`const`-`T` class.

  ```c++
  // Test smart pointers for nullness
  template <class T>
  class SmartPtr {
   public:
    // Could work, but ...
    operator void*();
  };
  
  class TreeNode;
  class Apple;
  class Orange;
  
  SmartPtr<TreeNode> ptn;
  
  if (ptn == 0)  // now fine
  if (ptn)  // also fine
  if (!ptn)  // fine
  
  SmartPtr<Apple> pa;
  SmartPtr<Orange> po;
  
  if (pa == po);  // this compiles!
  
  template <class T>
  class SmartPtr {
   public:
    // Much better
    bool operator!() const;
  };
  
  if (!ptn) {
    // ...
  } else {
    // ...
  }
  ```

  ```c++
  // Smart pointers and const
  template <class T>
  class SmartPtrToConst {
    // ...
   protected:
    union {
      const T* constPointee;
      T* pointee;
    };
  };
  
  template <class T>
  class SmartPtr : public SmartPtrToConst<T> {
    // ...
  };
  
  class CD;
  SmartPtr<CD> pCD = new CD("Famous Movie Themes");
  SmartPtrToConst<CD> pConstCD = pCD;  // fine
  ```

**Item 29: Reference counting**

- Reference counting is most useful for improving efficiency under the following conditions.

  - Relatively few values are shared by relatively many objects.
  - Object values are expensive to create or destroy, or they use lots of memory.

  ```c++
  // base class for reference-counted objects
  class RCObject {
  public:
    void addReference();
    void removeReference();
    void markUnshareable();
    bool isShareable() const;
    bool isShared() const;

  protected:
    RCObject();
    RCObject(const RCObject& rhs);
    RCObject& operator=(const RCObject& rhs);
    virtual ~RCObject();

  private:
    int refCount;
    bool shareable;
  };

  RCObject::RCObject() : refCount(0), shareable(true) {}

  RCObject::RCObject(const RCObject&) : refCount(0), shareable(true) {}

  RCObject& RCObject::operator=(const RCObject&) { return *this; }

  RCObject::~RCObject() {}

  void RCObject::addReference() { ++refCount; }

  void RCObject::removeReference() {
    if (--refCount == 0) {
      delete this;
    }
  }

  void RCObject::markUnshareable() { shareable = false; }

  bool RCObject::isShareable() const { return shareable; }

  bool RCObject::isShared() const { return refCount > 1; }
  ```

  ```c++
  // template class for smart pointers-to-T objects
  // T must inherit from RCObject
  template <class T>
  class RCPtr {
  public:
    RCPtr(T* realPtr = 0);
    RCPtr(const RCPtr& rhs);
    ~RCPtr();
    RCPtr& operator=(const RCPtr& rhs);
    T* operator->() const;
    T& operator*() const;

  private:
    T* pointee;
    void init();
  };

  template <class T>
  void RCPtr<T>::init() {
    if (pointee == 0) {
      return;
    }
    if (pointee->isShareable() == false) {
      pointee = new T(*pointee);
    }
    pointee->addReference();
  }

  template <class T>
  RCPtr<T>::RCPtr(T* realPtr) : pointee(realPtr) {
    init();
  }

  template <class T>
  RCPtr<T>::RCPtr(const RCPtr& rhs) : pointee(rhs.pointee) {
    init();
  }

  template <class T>
  RCPtr<T>::~RCPtr() {
    if (pointee) {
      pointee->removeReference();
    }
  }

  template <class T>
  RCPtr<T>& RCPtr<T>::operator=(const RCPtr& rhs) {
    if (pointee != rhs.pointee) {
      if (pointee) {
        pointee->removeReference();
      }
      pointee = rhs.pointee;
      init();
    }
    return *this;
  }

  template <class T>
  T* RCPtr<T>::operator->() const {
    return pointee;
  }

  template <class T>
  T& RCPtr<T>::operator*() const {
    return *pointee;
  }
  ```

  ```c++
  // class to be used by application developers
  class String {
  public:
    String(const char* value = "");
    char operator[](int index) const;
    char& operator[](int index);

  private:
    struct StringValue : public RCObject {
      char* data;
      StringValue(const char* initValue);
      StringValue(const StringValue& rhs);
      void init(const char* initValue);
      ~StringValue();
      friend class RCPtr<StringValue>;
    };
    RCPtr<StringValue> value;
  };

  void String::StringValue::init(const char* initValue) {
    data = new char[strlen(initValue) + 1];
    strcpy(data, initValue);
  }

  String::StringValue::StringValue(const char* initValue) { init(initValue); }

  String::StringValue::StringValue(const StringValue& rhs) { init(rhs.data); }

  String::StringValue::~StringValue() { delete[] data; }

  String::String(const char* initValue) : value(new StringValue(initValue)) {}

  char String::operator[](int index) const { return value->data[index]; }

  char& String::operator[](int index) {
    if (value->isShared()) {
      value = new StringValue(value->data);
    }
    value->markUnshareable();
    return value->data[index];
  }
  ```

**Item 30: Proxy classes**

- Objects that stand for other objects are often called proxy objects, and the classes that give rise to proxy objects are often called proxy classes.

- Taking the address of a proxy class yields a different type of pointer than does taking the address of a real object.

  ```c++
  class String {
   public:
    class CharProxy {
     public:
      CharProxy(String& str, int index);           // creation
      CharProxy& operator=(const CharProxy& rhs);  // lvalues uses
      CharProxy& operator=(char c);                // lvalues uses
      operator char() const;                       // rvalue uses
  
     private:
      String& theString;
      int charIndex;
    };
  
    const CharProxy operator[](int index) const;  // for const Strings
    CharProxy operator[](int index);              // for non-const Strings
    // ...
  
    friend class CharProxy;
  
   private:
    RCPtr<StringValue> value;
  };
  
  const String::CharProxy String::operator[](int index) const {
    return CharProxy(const_cast<String&>(*this), index);
  }
  
  String::CharProxy String::operator[](int index) { return CharProxy(*this, index); }
  
  String::CharProxy::CharProxy(String& str, int index) : theString(str), charIndex(index) {}
  
  // Because this function returns a character by value, and because C++ limits the use of such
  // by-value returns to rvalue contexts only, this conversion function can be used only in places
  // where an rvalue is legal.
  String::CharProxy::operator char() const { return theString.value->data[charIndex]; }
  
  // Move the code implementing a write into CharProxy's assignment operators, and that allows us to
  // avoid paying for a write when the non-const operator[] is used only in an rvalue context.
  String::CharProxy& String::CharProxy::operator=(const CharProxy& rhs) {
    if (theString.value->isShared()) {
      theString.value = new StringValue(theString.value->data);
    }
    theString.value->data[charIndex] = rhs.theString.value->data[rhs.charIndex];
    return *this;
  }
  
  String::CharProxy& String::CharProxy::operator=(char c) {
    if (theString.value->isShared()) {
      theString.value = new StringValue(theString.value->data);
    }
    theString.value->data[charIndex] = c;
    return *this;
  }
  ```

**Item 31: Making functions virtual with respect to more than one object**

- The most common approach to double-dispatching is via chains of `if-then-else`s.

- To minimize the risks inherent in an RTTI approach, the strategy is to implement double-dispatching as two single dispatches.

- Use a vtbl to eliminate the need for compilers to perform chains of `if-then-else`-like computations, and it allows compilers to generate the same code at all virtual function call sites.

- The recompilation problem would go away if our associative array contained pointers to non-member functions.

  - Everything in an unnamed namespace is private to the current translation unit (essentially the current file) -- it's just like the functions were declared `static` at file scope.

  ```C++
  // Use virtual functions and RTTI
  class GameObject {
   public:
    virtual void collide(GameObject& otherObject) = 0;
    // ...
  };
  
  class SpaceShip : public GameObject {
   public:
    virtual void collide(GameObject& otherObject);
    // ...
  };
  
  class SpaceStation : public GameObject {
   public:
    virtual void collide(GameObject& otherObject);
    // ...
  };
  
  class Asteroid : public GameObject {
   public:
    virtual void collide(GameObject& otherObject);
    // ...
  };
  
  class CollisionWithUnknownObject {
   public:
    CollisionWithUnknownObject(GameObject& whatWeHit);
    // ...
  };
  
  void SpaceShip::collide(GameObject& otherObject) {
    const type_info& objectType = typeid(otherObject);
    if (objectType == typeid(SpaceShip)) {
      SpaceShip& ss = static_cast<SpaceShip&>(otherObject);
      // process a SpaceShip-SpaceShip collision
    } else if (objectType == typeid(SpaceStation)) {
      SpaceStation& ss = static_cast<SpaceStation&>(otherObject);
      // process a SpaceShip-SpaceStation collision
    } else if (objectType == typeid(Asteroid)) {
      Asteroid& ss = static_cast<Asteroid&>(otherObject);
      // process a SpaceShip-Asteroid collision
    } else {
      throw CollisionWithUnknownObject(otherObject);
    }
  }
  ```

  ```c++
  // Use virtual functions only
  class SpaceShip;
  class SpaceStation;
  class Asteroid;
  
  class GameObject {
   public:
    virtual void collide(GameObject& otherObject) = 0;
    virtual void collide(SpaceShip& otherObject) = 0;
    virtual void collide(SpaceStation& otherObject) = 0;
    virtual void collide(Asteroid& otherObject) = 0;
    // ...
  };
  
  class SpaceShip : public GameObject {
   public:
    virtual void collide(GameObject& otherObject);
    virtual void collide(SpaceShip& otherObject);
    virtual void collide(SpaceStation& otherObject);
    virtual void collide(Asteroid& otherObject);
    // ...
  };
  
  // Compilers figure out which of a set of functions to call on the basis of the static types of the
  // arguments passed to the function.
  void SpaceShip::collide(GameObject& otherObject) { otherObject.collide(*this); }
  ```

  ```c++
  // Emulate virtual function tables
  class GameObject {
   public:
    virtual void collide(GameObject& otherObject) = 0;
    // ...
  };
  
  class SpaceShip : public GameObject {
   public:
    virtual void collide(GameObject& otherObject);
    virtual void hitSpaceShip(GameObject& otherObject);
    virtual void hitSpaceStation(GameObject& otherObject);
    virtual void hitAsteroid(GameObject& otherObject);
    // ...
  
   private:
    typedef void (SpaceShip::*HitFunctionPtr)(GameObject&);
    typedef std::map<std::string, HitFunctionPtr> HitMap;
    HitFunctionPtr lookup(const GameObject& whatWeHit) const;
    static HitMap* initializeCollisionMap();
    // ...
  };
  
  void SpaceShip::hitSpaceShip(GameObject& spaceShip) {
    SpaceShip& otherShip = dynamic_cast<SpaceShip&>(spaceShip);
    // ...
  }
  
  void SpaceShip::hitSpaceShip(GameObject& spaceShip) {
    SpaceShip& otherShip = dynamic_cast<SpaceShip&>(spaceShip);
    // ...
  }
  
  void SpaceShip::collide(GameObject& otherObject) {
    HitFunctionPtr hfp = lookup(otherObject);
    if (hfp) {
      (this->*hfp)(otherObject);
    } else {
      throw CollisionWithUnknownObject(otherObject);
    }
  }
  
  SpaceShip::HitFunctionPtr SpaceShip::lookup(const GameObject& whatWeHit) const {
    static std::unique_ptr<HitMap> collisionMap(initializeCollisionMap());
    HitMap::iterator mapEntry = collisionMap->find(typeid(whatWeHit).name());
    if (mapEntry == collisionMap->end()) {
      return 0;
    }
    return (*mapEntry).second;
  }
  
  SpaceShip::HitMap* SpaceShip::initializeCollisionMap() {
    HitMap* phm = new HitMap;
    (*phm)["SpaceShip"] = &hitSpaceShip;
    (*phm)["SpaceStation"] = &hitSpaceStation;
    (*phm)["Asteroid"] = &hitAsteroid;
    return phm;
  }
  ```

## CH6: Miscellany

**Item 32: Program in the future tense**

- Provide complete classes, even if some parts aren't currently used. When new demands are made on your classes, you're less likely to have to go back and modify them.

- Design your interfaces to facilitate common operations and prevent common errors. Make the classes easy to use correctly, hard to use incorrectly.

- If there is no great penalty for generalizing your code, generalize it.

**Item 33: Make non-leaf classes abstract**

- Non-leaf classes should be abstract. Adherence to it will yield dividends in the form of increased reliability, robustness, comprehensibility, and extensibility throughout your software.

  ```c++
  // Wrong: partial assignment
  class Animal {
   public:
    Animal& operator=(const Animal& rhs);
    // ...
  };
  
  class Lizard : public Animal {
   public:
    Lizard& operator=(const Lizard& rhs);
    // ...
  };
  
  class Chicken : public Animal {
   public:
    Chicken& operator=(const Chicken& rhs);
    // ...
  };
  
  Lizard liz1;
  Lizard liz2;
  Animal* pAnimal1 = &liz1;
  Animal* pAnimal2 = &liz2;
  // Only the Animal part of liz1 will be modified
  *pAnimal1 = *pAnimal2;
  ```

  ```c++
  // The goal is to identify useful abstractions and to force them - and only them - into existence as
  // abstract classes
  class AbstractAnimal {
   protected:
    AbstractAnimal& operator=(const AbstractAnimal& rhs);
  
   public:
    virtual ~AbstractAnimal() = 0;
    // ...
  };
  
  class Animal : public AbstractAnimal {
   public:
    Animal& operator=(const Animal& rhs);
    // ...
  };
  
  class Lizard : public AbstractAnimal {
   public:
    Lizard& operator=(const Lizard& rhs);
    // ...
  };
  
  class Chicken : public AbstractAnimal {
   public:
    Chicken& operator=(const Chicken& rhs);
    // ...
  };
  ```

**Item 34: Understand how to combine C++ and C in the same program**

- Make sure the C++ and C compilers produce compatible object files.

- Declare functions to be used by both languages `extern "C"`.

- If at all possible, write `main` in C++.

- Always use `delete` with memory from `new`; always use `free` with memory from `malloc`.

- Limit what you pass between the two languages to data structures that compile under C; the C++ version of structs may contain non-virtual member functions.

  ```C++
  // Name mangling
  #ifdef __cplusplus
  extern "C" {
  #endif
  void drawLine(int x1, int y1, int x2, int y2);
  void twiddleBits(unsigned char bits);
  void simulate(int iterations);
  // ...
  #ifdef __cplusplus
  }
  #endif
  ```

**Item 35: Familiarize yourself with the language standard**

- New features have been added.

- Templates have been extended.

- Exception handling has been refined.

- Memory allocation routines have been modified.

- New casting forms have been added.

- Language rules have been refined.

- Support for the standard C library.

- Support for strings.

- Support for localization.

- Support for I/O.

- Support for numeric applications.

- Support for general-purpose containers and algorithms.
