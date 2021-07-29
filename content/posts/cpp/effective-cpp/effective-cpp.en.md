---
title: "[Notes] Effective C++, Third Edition"
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

## Ch. 1: Accustoming Yourself to C++

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

## Ch. 2: Constructors, Destructors, and Assignment Operators

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

## Ch. 3: Resource Management

**Item 13: Use objects to manage resources.**

- To prevent resource leaks, use Resource Acquisition Is Initialization (RAII) objects that acquire resources in their constructors and release them in their destructors.
- Two commonly useful RAII classes are `tr1::shared_ptr` and `auto_ptr`. `tr1::shared_ptr` is usually the better choice, because its behavior when copied is intuitive. Copying an `auto_ptr` sets it to null.

**Item 14: Think carefully about copying behavior in resource-managing classes.**

- Copying an RAII object entails copying the resource it manages, so the copying behavior of the resources determines the copying behavior of the RAII object.
  - Solution 1: prohibit copying.
  - Solution 2: reference-count the underlying resource.
  - Solution 3: copy the underlying resource.
  - Solution 4: transfer ownership of the underlying resource.

**Item 15: Provide access to raw resources in resource-managing classes.**

- APIs often require access to raw resources, so each RAII class should offer a way to get at the resource it manages.
- Access may be via explicit conversion or implicit conversion. In general, explicit conversion is safer, but implicit conversion is more convenient for clients.

**Item 16: Use the same form in corresponding uses of new and delete.**

- If you use [] in a `new` expression, you must use [] in the corresponding `delete` expression. If you don't use [] in a `new` expression, you mustn't use [] in the corresponding `delete` expression.

**Item 17: Store `new`ed objects in smart pointers in standalone statements.**

- Store `new`ed objects in smart pointers in standalone statements. Failure to do this can lead to subtle resource leaks when exceptions are throw.
  - An exception can intervene between the time a resource is created and the time that resource is turned over to a resource-managing object.

## Ch. 4: Designs and Declarations

**Item 18: Make interfaces easy to use correctly and hard to use incorrectly.**

- Good interfaces are easy to use correctly and hard to use incorrectly. You should strive for these characteristics in all your interfaces.
- Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.
- Ways to prevent errors include creating new types, restricting operations on types, constraining object values, and eliminating client resource management responsibilities.
- `tr1::shared_ptr` supports custom deleters. This prevents the cross-DLL problem, can be used to automatically unlock mutexes, etc.

**Item 19: Treat class design as type design.**

- Class design is type design. Before defining a new type, be sure to consider all the issues discussed in this item.
  - How should objects of your new type be created and destroyed?
  - How should object initialization differ from object assignment?
  - What does it mean for objects of your new type to be passed by value?
  - What are the restrictions on legal values for your new type?
  - Does your new type fit into an inheritance graph?
  - What kind of type conversions are allowed for your new type?
  - What operators and functions make sense for the new type?
  - What standard functions should be disallowed?
  - Who should have access to the members of your new type?
  - What is the "undeclared interface" of your new type?
  - How general is your new type?
  - Is a new type really what you need?

**Item 20: Prefer pass-by-reference-to-`const` to pass-by-value.**

- Prefer pass-by-reference-to-`const` over pass-by-value. It's typically more efficient and it avoids the slicing problem.
- The rule doesn't apply to built-in types and STL iterator and function object types. For them, pass-by-value is usually appropriate.

**Item 21: Don't try to return a reference when you must return an object.**

- Never return  a pointer or reference to a local stack object, a reference to a heap-allocated object, or a pointer or reference to a local static object if there is a chance that more than one such object will be needed.

**Item 22: Declare data members `private`.**

- Declare data members `private`. It gives clients syntactically uniform access to data, affords fine-grained access control, allows invariants to be enforced, and offers class authors implementation flexibility.
- `protected` is no more encapsulated than `public`.

**Item 23: Prefer non-member non-friend functions to member functions.**

- Prefer non-member non-friend functions to member functions. Doing so increases encapsulation, packaging flexibility, and functional extensibility.

**Item 24: Declare non-member functions when type conversions should apply to all parameters.**

- If you need type conversions on all parameters to a function (including the one that would otherwise be pointed to by the `this` pointer), the function must be a non-member.

**Item 25: Consider support for a non-throwing `swap`.**

- Provide a `swap` member function when `std::swap` would be inefficient for your type. Make sure your `swap` doesn't throw exceptions.
- If you offer a member `swap`, also offer a non-member `swap` that calls the member. For classes (not templates), specialize `std::swap`, too.
  ```c++
  class WidgetImpl;
  
  class Widget {
   public:
    // ...
    void swap(Widget& other) {
      using std::swap;
      swap(plmpl, other.plmpl);
    }
    // ...
  
   private:
    WidgetImpl* plmpl;
  };
  
  namespace std {
  template <>
  void swap<Widget>(Widget& a, Widget& b) {
    a.swap(b);
  }
  }  // namespace std
  ```
- When calling `swap`, employ a `using` declaration for `std::swap`, then call `swap` without namespace qualification.
  ```c++
  template <typename T>
  void doSomething(T& obj1, T& obj2) {
    using std::swap;
    // ...
    swap(obj1, obj2);
    // ...
  }
  ```
- It's fine to totally specialize `std` templates for user-defined types, but never try to add something completely new to `std`.
  - Though C++ allows partial specialization of class templates, it doesn't allow it for function templates.
    ```c++
    // invalid code
    namespace std {
    template <typename T>
    void swap(Widget<T>& a, Widget<T>& b) {
      a.swap(b);
    }
    }  // namespace std
    ```

## Ch. 5: Implementations

**Item 26: Postpone variable definitions as long as possible.**

- Postpone variable definitions as long as possible. It increases program clarity and improves efficiency.
- Unless you know the following two points, you should default to defining the variable used only inside the loop.
  - Assignment is less expensive than a constructor-destructor pair.
  - Dealing with a performance-sensitive part of your code.
  ```c++
  for (int i = 0; i < n; ++i) {
    Widget w(some value dependent on i);
    // ...
  }
  ```

**Item 27: Minimize casting.**

- Avoid casts whenever practical, especially dynamic_casts in performance-sensitive code. If a design requires casting, try to develop a cast-free alternative.
  - Use containers that store pointers to derived class objects directly, thus eliminating the need to manipulate such objects through base class interfaces.
    ```c++
    typedef std::vector<std::tr1::shared_ptr<SpecialWindow>> VPSW;
    VPSW winPtrs;
    // ...
    for (VPSW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
      (*iter)->blink();
    }
    ```
  - Provide virtual functions in the base class that let you do what you need.
    ```c++
    class Window {
     public:
      virtual void blink() {}
      // ...
    };
    
    class SpecialWindow : public Window {
     public:
      virtual void blink() {}
      // ...
    };
    ```
- When casting is necessary, try to hide it inside a function. Clients can then call the function instead of putting casts in their own code.
- Prefer C++-style casts to old-style casts. They are easier to see, and they are more specific about what they do.
  ```c++
  // C-style casts look like this:
  (T)expression
  // Function-style casts use this syntax:
  T(expression)
  
  // C++-style casts look like this:
  const_cast<T>(expression)
  dynamic_cast<T>(expression)
  reinterpret_cast<T>(expression)
  static_cast<T>(expression)
  ```

**Item 28: Avoid returning "handles" to object internals.**

- Avoid returning handles (references, pointers, or iterators) to object internals. Not returning handles increases encapsulation, helps `const` member functions act `const`, and minimizes the creation of dangling handles.
  - Dangling handles are handles that refer to parts of objects that don't exist any longer.
    ```c++
    class Rectangle;
    class GUIObject;
    
    const Rectangle boundingBox(const GUIObject& obj);
    
    GUIObject* pgo;
    // at the end of the statement, boundingBox's return value - temp - will be
    // destroyed, and that will indirectly lead to the destruction of temp's Points
    const Point* pUpperLeft = &(boundingBox(*pgo).upperLeft());
    ```

**Item 29: Strive for exception-safe code.**

- Exception-safe functions leak no resources and allow no data structures to become corrupted, even when exceptions are thrown.
  - Functions offering the basic guarantee promise that if an exception is thrown, everything in the program remains in a valid state.
  - Functions offering the strong guarantee promise that if an exception is thrown, the state of the program is unchanged.
  - Functions offering the nothrow guarantee promise never to throw exceptions.
- The strong guarantee can often be implemented via copy-and-`swap`, but the strong guarantee is not practical for all functions.
- A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls.

**Item 30: Understand the ins and outs of inlining.**

- Limit most inlining to small, frequently called functions. This facilitates debugging and binary upgradability, minimizes potential code bloat, and maximizes the chances of greater program speed.
- Don't declare function templates `inline` just because they appear in header files.

**Item 31: Minimize compilation dependencies between files.**

- The general idea behind minimizing compilation dependencies is to depend on declarations instead of definitions. Two approaches based on this idea are Handle classes and Interface classes.
  - Classes like `Person` that employ the pimpl idiom are often called Handle classes. One way is to forward all their function calls to the corresponding implementation classes and have those classes do the real work.
    ```c++
    #include "Person.h"
    #include "PersonImpl.h"
    
    Person::Person(const std::string& name, const Data& birthday, const Address& addr)
        : pImpl(new PersonImpl(name, birthday, addr)) {}
    
    std::string Person::name() const { return pImpl->name(); }
    ```
  - Clients of an Interface classes must have a way to create new objects. They typically do it by calling a function that plays the role of the constructor for the derived classes that are actually instantiated. Such functions are typically called factory functions or virtual constructors.
    ```c++
    // Interface class:
    class Person {
     public:
      static std::shared_ptr<Person> create(const std::string& name, const Date& birthday,
                                            const Address& addr);
      virtual ~Person();
    
      virtual std::string name() const = 0;
      virtual std::string birthDate() const = 0;
      virtual std::string address() const = 0;
      // ...
    };
    
    std::shared_ptr<Person> Person::create(const std::string& name, const Date& birthday,
                                           const Address& addr) {
      return std::shared_ptr<Person>(new RealPerson(name, birthday, addr));
    }
    
    // Clients use them like this:
    int main() {
      std::string name;
      Date dateOfBirth;
      Address address;
    
      std::shared_ptr<Person> pp(Person::create(name, dateOfBirth, address));
      // ...
      std::cout << pp->name() << "was born on" << pp->birthDate() << " and now lives at"
                << pp->address();
      // ...
    }
    ```
- Library header files should exist in full and declaration-only forms. This applies regardless of whether templates are involved.

## Ch. 6: Inheritance and Object-Oriented Design

**Item 32: Make sure public inheritance models "is-a."**

- Public inheritance means "is-a". Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object.

**Item 33: Avoid hiding inherited names.**

- Names in derived classes hide names in base classes. Under public inheritance, this is never desirable.
- To make hidden names visible again, employ using declarations or forwarding functions.
  ```c++
  // with using declarations:
  class Base {
   private:
    int x;
  
   public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
    // ...
  };
  
  class Derived : public Base {
   public:
    using Base::mf1;
    using Base::mf3;
    virtual void mf1();
    void mf3();
    void mf4();
    // ...
  };
  
  // with forwarding functions:
  class Base {
   public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    // ...
  };
  
  class Derived : private Base {
   public:
    virtual void mf1() { Base::mf1(); }
    // ...
  };
  ```
- Another use for `inline` forwarding functions is to work around ancient compilers that (incorrectly) don't support `using` declarations to import inherited names into the scope of a derived class.

**Item 34: Differentiate between inheritance of interface and inheritance of implementation.**

- Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes always inherit base class interfaces.
- Pure virtual functions specify inheritance of interface only.
- Simple (impure) virtual functions specify inheritance of interface plus inheritance of a default implementation.
- Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation.

**Item 35: Consider alternatives to virtual functions.**

- Alternatives to virtual functions include the non-virtual interface idiom (NVI idiom) and various forms of the Strategy design pattern. The NVI idiom is itself an example of the Template Method design pattern.
- A disadvantage of moving functionality from a member function to a function outside the class is that the non-member function lacks access to the class's non-public members.
- `tr1::function` objects act like generalized function pointers. Such objects support all callable entities compatible with a given target signature.

**Item 36: Never redefine an inherited non-virtual function.**

- Never redefine an inherited non-virtual function.

**Item 37: Never redefine a function's inherited default parameter value.**

- Never redefine an inherited default parameter value, because default parameter values are statically bound, while virtual functions - the only functions you should be redefining - are dynamically bound.

**Item 38: Model "has-a" or "is-implemented-in-terms-of" through composition.**

- Composition has meanings completely different from that of public inheritance.
- In the application domain, composition means has-a. In the implementation domain, it means is-implemented-in-terms-of.
  ```c++
  // "has-a"
  class Address {};
  class PhoneNumber {};
  
  class Person {
   public:
    // ...
      
   private:
    std::string name;
    Address address;
    PhoneNumber voiceNumber;
    PhoneNumber faxNumber;
  };
  
  // "is-implemented-in-terms-of"
  template <class T>
  class Set {
   public:
    bool member(const T& item) const;
    void insert(const T& item);
    void remove(const T& item);
    std::size_t size() const;
  
   private:
    std::list<T> rep;
  };
  ```

**Item 39: Use private inheritance judiciously.**

- Private inheritance means is-implemented-in-terms of. It's usually inferior to composition, but it makes sense when a derived class needs access to protected base class members or needs to redefine inherited virtual functions.
  ```c++
  class Timer {
   public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
    // ...
  };
  
  class Widget : private Timer {
   private:
    virtual void onTick() const;
    // ...
  };
  ```
- Unlike composition, private inheritance can enable the empty base optimization, This can be important for library developers who strive to minimize object sizes.

**Item 40: Use multiple inheritance judiciously.**

- Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance.
  ```c++
  class BorrowableItem {
   public:
    void checkOut();
    // ...
  };
  
  class ElectronicGadget {
   private:
    bool checkOut();
    // ...
  };
  
  class MP3Player : public BorrowableItem, public ElectronicGadget {};
  
  MP3Player mp;
  mp.checkOut();                  // ambiguous
  mp.BorrowableItem::checkOut();  // ah, that checkOut...
  ```
- Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It's most practical when virtual base classes have no data.
  - C++ performs the replication by default when it comes to "deadly MI diamond".
  - The responsibility for initializing a virtual base is borne by the most derived class in the hierarchy.
- Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.

## Ch. 7: Templates and Generic Programming

**Item 41: Understand implicit interfaces and compile-time polymorphism.**

- Both classes and templates support interfaces and polymorphism.
- For classes, interfaces are explicit and centered on function signatures. Polymorphism occurs at runtime through virtual functions.
- For template parameters, interfaces are implicit and based on valid expressions. Polymorphism occurs during compilation through template instantiation and function overloading resolution.
  ```c++
  class Class1 {
   public:
    void interfaceFunc();
    void otherFunc1();
  };
  
  class Class2 {
   public:
    void interfaceFunc();
    void otherFunc2();
  };
  
  // implicit interfaces:
  template <typename T>
  class UseClass {
   public:
    void run(T& obj) { obj.interfaceFunc(); }
  };
  ```

**Item 42: Understand the two meanings of `typename`.**

- When declaring template parameters, `class` and `typename` are interchangeable.
- If the parser encounters a nested dependent name in a template, it assumes that the name is not a type unless you tell it otherwise.
  - Names in a template that are dependent on a template parameter are called dependent names.
  ```c++
  template <typename C>
  void print2nd(const C& container) {
    if (container.size() >= 2) {
      typename C::const_iterator iter(container.begin());
      // ...
    }
  }
  ```
- Use `typename` to identify nested dependent type names, except in base class lists or as a base class identifier in a member initialization list.
  ```c++
  template <typename T>
  class Derived : public Base<T>::Nested {
   public:
    explicit Derived(int x) : Base<T>::Nested(x) {
      typename Base<T>::Nested temp;
      // ...
    }
  };
  ```

**Item 43: Know how to access names in templatized base classes.**

- In derived class templates, refer to names in base class templates via a "`this->`" prefix, via `using` declarations, or via an explicit base class qualification.

**Item 44: Factor parameter-independent code out of templates.**

- Templates generate multiple classes and multiple functions, so any template code not dependent on a template parameter causes bloat.
- Bloat due to non-type template parameters can often be eliminated by replacing template parameters with function parameters or class data members.
- Bloat due to type parameters can be reduced by sharing implementations for instantiation types with identical binary representations.

**Item 45: Use member function templates to accept "all compatible types."**

- Use member function templates to generate functions that accept all compatible types.
  ```c++
  template <typename T>
  class SmartPtr {
   public:
    template <typename U>
    SmartPtr(const SmartPtr<U>& other) : heldPtr(other.get()) {
      // ...
    }
  
    T* get() const {return heldPtr};
  
   private:
    T* heldPtr;
  };
  ```
- If you declare member templates for generalized copy construction or generalized assignment, you'll still need to declare the normal copy constructor and copy assignment operator, too.
  ```c++
  template <class T>
  class shared_ptr {
   public:
    shared_ptr(shared_ptr const& r);
    template <class Y>
    shared_ptr(shared_ptr<Y> const& r);
    shared_ptr& operator=(shared_ptr const& r);
    template <class Y>
    shared_ptr& operator=(shared_ptr<Y> const& r);
    // ...
  };
  ```

**Item 46: Define non-member functions inside templates when type conversions are desired.**

- Implicit type conversion functions are never considered during template argument deduction.
- Class templates don't depend on template argument deduction (that process applies only to function templates).
- When writing a class template that offers functions related to the template that support implicit type conversions on all parameters, define those functions as friends inside the class template.
  ```c++
  template <typename T>
  class Rational {
   public:
    // ...
    friend const Rational operator*(const Rational& lhs, const Rational& rhs);
    // inside a class template, the name of the template can be used as shorthand for
    // the template and its parameters
    // friend const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs);
  };
  
  template <typename T>
  const Rational<T> operator*(const Rational<T>& lhs, Rational<T>& rhs) {
    // ...
  }
  ```

**Item 47: Use traits classes for information about types.**

- Traits classes make information about types available during compilation. They're implemented using templates and template specializations.
- In conjunction with overloading, traits classes make it possible to perform compile-time `if...else` tests on types.

**Item 48: Be aware of template metaprogramming.**

- Template meta programming (TMP) can shift work from runtime to compile-time, thus enabling earlier error detection and higher runtime performance.
  - Compilers are obliged to make sure that all source code is valid, even if it's not executed.
- TMP can be used to generate custom code based on combinations of policy choices, and ti can also be used to avoid generating code inappropriate for particular types.

## Ch. 8: Customizing new and delete

**Item 49: Understand the behavior of the new-handler.**

**Item 50: Understand when it makes sense to replace new and delete.**

**Item 51: Adhere to convention when writing new and delete.**

**Item 52: Write placement delete if you write placement new.**

## Ch. 9: Miscellany

**Item 53: Pay attention to compiler warnings.**

**Item 54: Familiarize yourself with the standard library, including TR1.**

**Item 55: Familiarize yourself with Boost.**
