Any expression in C++ can fall into one of a small number of categories called **value categories** of which the two most important are **rvalues** and **lvalues**. All expressions are either an rvalue or lvalue. This terminology is often criticized, and rightly so. First of all, they should be called "expression categories", as they categorize expressions, not values, and we should have "l-expression" and "r-expression" categories instead of rvalue and lvalue. Furthermore, lvalue and rvalue brings to mind "left value" and "right values", and you might think there may be some significance there. The sensible analogy to "left" and "right" is that lvalues and rvalues are opposites; an expression cannot be both an rvalue and an lvalue at the same time. Trying to make any further connection between leftness/rightness and lvalue/rvalue only leads to confusion.

To discuss the specifics of value categories with we will first present their historical evolution.

## Before C++11

Two defining characteristics of expressions are **value**, which is the actual value the expression computes, and an **address** which is the physical location in memory where the value is stored. (Expressions in C++ also have **types** but that is not signficant for this discussion.)  Despite what the name suggests, a "value category" tells us something about the __address__ of any given expression:

 - lvalues were expressions whose addresses are guaranteed to made available to the programmer. Hence, we will say that lvalues are **locatable**.
 - There is no guarantee the address of an rvalues will be made available to the programmer. Hence, we will say that rvalues are **latent**.

Some examples will elucidate:

```c++
int x = 3;
x; // this is an lvalue
```
 - the data stored in `x` is accessible to us with a named variable. in general, names of things represent locatable data in C++. This includes names of objects, functions, etc. Since the data is stored in a variable, the line of code after the expression `x` still has access to the location in memory which stores `x`'s data. This is why `&` is defined for `x`, as it is safe to reveal this address to the programmer. That location in memory will have the data in `x` as long as `x` remains in scope.

```c++
int a;
int b;

// do stuff; assume a and b get assigned something 

if (a + b == 3) {   // <-- "a + b" is an rvalue
  // do stuff
};
```
 - When we write `a + b`, the result of that computation  is whipped up and represented somewhere, but since it is not stored in a variable, it will not be made accessible to the programmer in future lines of code. The compiler is allowed to trash the value at some indeterminate machine instruction sometime after this line of code. While we can get the same *value* by writing `a + b` again in the next line of code, this data will be represented in a new address in memory. Therefore, it is unsafe to reveal this address to the programmer, and for the programmer's safety, the address is made latent.

```c++
 - int& number_reference = 3; // '3' here is an rvalue.
```
 - Integer literals are, in general, rvalues. While rvalues are latent, their address can be saved __if__ they stored into a reference. In this case, the location in memory for `3` will be referenced by `number_reference` and the lifetime of the value for the literal is extended such that it shares the lifetime of the reference. When we find the address of `number_reference` we find the address of the value that was represented by the expression `3`. That is, `3`'s address is made available indirectly through the reference.

<details>
<summary>Note</summary>
<br>
Some people call rvalues "temporary values", and unfortunately this sort of terminology has even permeated the C++ standard. But it is clear from the last example that rvalues are not guaranteed to "immediately vanish" by the next line of code. The terminology "temporary" is objectively incorrect so I am avoiding it entirely in this discussion.
</details>

<details>
<summary>Note</summary>
<br>
It is a common misconception that lvalues and rvalues indicate the lifetime of data; one might think that lvalues are expressions whose data has persistent lifetime, and rvalues are expressions whose data has temporary lifetime. The third example shows why rvalues are not, in general, temporary. I regret to inform that some abuses of the language specification make it possible to create lvalues that refer to data that is no longer in scope. The lesson is clear: in general, do not associate value category with lifetime.
</details>

## C++11

This sections assumes you have some familiarity with C++ move semantics. If not, please open the following dropdown.

<details>
<summary>C++ move semantics</summary>

We use the term **resource** to describe something that must be freed in an exception-safe way. Resources are things that in any other language you would manage with a try-catch-finally block (things like allocated memory, an open file, a lock, a connection to a socket, etc). Since C++ does not have a finally block, the "proper" way to safely manage resources is to create classes whose instantiated objects manage a single resource. The constructor acquires the resource and the destructor frees the resource. Since a destructor is called whenever an object goes out of scope, a destructor is guaranteed to be called even if an exception occurs. This design pattern is so important that is has a name: RAII (Resource Acquisition Is Initialization).

It can be useful to exchange ownership of a resource from one object to another. This concept is called a **move**. If object B represents acquired memory allocation and wants to exchange ownership of this allocated memory to object A, the allocated memory is represented as a field whose value is a pointer to that memory. The move will assign this pointer to the appropriate field A and make sure B now has a `nullptr` in this field, indicating that it no longer owns a resource.

This concept is useful enough that the C++ standards committee wanted to add language semantics which allow for a stylistic standard for creating moves. They chose to implement this by adding a new type of reference. Constructors and assignment overloads could be written by the programmer which accept this new type of reference, by which it is understood by the programmer that references of this type given to a constructor or assignment are intended to be moved from. We can think of this new reference as a flag which indicates that the object it points to is intended to be moved from. This new reference is referred to as an `rvalue reference`, and the pre-C++`` reference was renamed to an `lvalue reference`. The reason the new reference was called an rvalue reference because if you have an overloaded function which takes an rvalue or lvalue reference, passing an rvalue to that function causes the rvalue reference signature to be called instead of the lvalue. This allows you to create a function which has different behavior depending on whether the argument provided is an rvalue or lvalue. The rvalue reference is also a mutable reference if declared non-const. Before C++11, rvalues could only be bound to `const` lvalue references meaning there was no way to mutate a reference bound to an rvalue.

The name "rvalue reference" is very poor because it suggests you can only bind rvalues to rvalue references. But you can bind an lvalue to an rvalue reference. This is a common point of confusion for new C++ developers.

The C++ standards committee added a function `std::move` to the library which takes a single argument and returns an rvalue reference to the object given to the function. This function should have been called `std::as_movable` since no move occurs in the execution of `std::move`, a common point of confusion among new C++ developers. Oh well.

We should interpret the return value of `std::move` as a flag to an object which indicates that the object can be moved from. If the user provides the result of `std::move` to a constructor or assignment which is overloaded to accept an rvalue reference, we cause a move if these overloads exist and are implemented correctly. A constructor of a class `T` which accepts a `T` rvalue reference parameter is called a **move constructor**. An assignment operator overload of class `T` which accepts a `T` rvalue reference parameter is called a **move assignment overload**.

</details>

In C++11, the standards committee generalized the concept of "locatibility" into what they called "identity" and wanted to categorize expressions whose evaluation corresponds to an object's identity. An object has "identity" if the evaluation of the expression offers some mechanism to the programmer to determine an object's uniqueness. Obviously, all "locatable" expressions (lvalues) have identity since the & operator can be used with them to find an object's identity, but an expression could also carry identity if its __value__ was guaranteed to identify an object. Such a thing could happen in an rvalue expression which also somehow computes a reference since the C++ specification requires that a reference actually points to something (unlike a pointer, which can point to `nullptr`). Such an expression manifested in the language with the *rvalue reference cast* introduced in C++11. Casting an object into an rvalue reference of object type results in latent data (if I don't store it in a variable the result of the cast won't be accessible in subsequent lines of code) but since it is a reference it is an alias to an object and guarantees us the identity of that object. The C++ standards committee felt this expression was best represented by a third value category, and the pre-existing value category taxonomy was salvaged by changing the meaning of the word "rvalue":
 - nearly all of what we used to call rvalues are now called **prvalues** ("pure rvalues"),
 - what we used to call lvalues are still called lvalues,
 - an __rvalue reference cast expression of an object type__ is of a new value category called **xvalue**,
   - __a function which returns an rvalue reference to an object type__ is also considered an xvalue,
 - and rvalue is now an umbrella term: __xvalues and prvalues are specific types of rvalues__.

There also exists the less useful umbrella term glvalue, short for "general lvalue", which refers to either xvalues or lvalues (yes, this means an xvalue is both a glvalue and an rvalue). Hence glvalues are all expressions which carry identity. As a result of this change, everything that was an lvalue before was still an lvalue and everything that used to be an rvalue was still an rvalue, but now there two specific and mutually-exclusive types of rvalues.

With this new change, a new constructor overload called a "move constructor" could be created for an object which takes an rvalue reference to object whose type is that of the class for that constructor. A move constructor is called if an rvalue or xvalue of the correct object type is given to the constructor, and unlike const lvalue references, rvalue references are mutable, meaning the move constructor can modify the object given to it in order to perform its task. (It is expected that the user implements a move constructor such that the new constructed object "moves" the resource(s) from the given object so that ownership of the memory/mutex/file/etc associated with the given object is exchanged with that of the newly constructed object. While this is a mere convention, it would be considered a mortal sin if your move constructor did anything other than this.) As such, rvalues are now considered "movable". lvalues have identity and are not movable, prvalues have no identity and are movable, and xvalues have identity and are movable.

The new term xvalue was originally introduced without any meaning. I prefer to think of xvalue as being short for "cross value" since an xvalue contains a cross of some characteristics from lvalues and some characteristics from pre-C++11 rvalues. There is a small class of expressions that were rvalues before C++11 that are now also xvalues, but the rvalue cast expression and the function which returns an rvalue reference are by far the most important and common examples encountered in practice so it will be what we will emphasize in the current discussion.

## C++17

C++17 added more wrinkles to the taxonomy. Before C++17, we could have a prvalue which represented a latent object (for example, a function call of a function whose return statement calls a class constructor). However, if a prvalue is of a class type, it no longer represents an object but instead acts as a "free coupon" for that result object. Hence there is no expensive object copy when a prvalue result is returned by a function (since C++ is a pass-by-value language), we simply "xerox the coupon" for the actual resultant object. The specification demands that, at some point, the object is materialized (the coupon is exchanged) into the actual result object, and which point the specification also demands that the prvalue is converted into an xvalue.

With this new feature, prvalues of nonprimitive types are no longer moved from. When we pass a prvalue to a class constructor, the prvalue is eventually materialized into an xvalue, and that xvalue is moved from. Hence in C++17 onwards, it is no longer true that prvalues are movable.

## Summary

In short:
 - the distinguishing factor between lvalues and rvalues is whether or not the address of the value of the expression is guaranteed to be available to the programmer
 - lvalues are locatable--their address is guaranteed to be made available to the programmer
 - rvalues are latent--their address is only made available through assignment which allows indirect access to the address
 - changes to the C++11 and C++17 do not change these facts, and only serve to introduce two specific types of rvalues (prvalues and xvalues)
 - after C++11 and before C++17,
   - lvalues have identity and cannot be moved
   - xvalues have identity and are movable
   - prvalues do not have identity and are movable
   - xvalues and prvalues are specific types of rvalues.
 - prvalues of class types were movable until C++17, in which only xvalues are allowed to represent movable objects of class types.

### An aside on the xvalue terminology

I will mention that the standard says that xvalues are "eXpiring values", terminology I strongly dislike. Since most practical usage of xvalues is to call a move constructor or move assignement, the value represented by the xvalue is meant to be "moved from" and eventually discarded, which I suppose what this terminology is supposed to mean. This hinges on the programmer adhering to convention, however:

 - The class type might not have a move constructor or move assignment overload. Then the xvalue won't be moved from at all.
 - The move constructor or move assignment overload could be implemented in a faulty or nefarious way which does not perform the expected behavior.
 - The user could have an lvalue pointing to some object, cast it as an xvalue and move it to some other variable, and then access the original lvalue and manipulate the object that was moved from. Such a thing would be considered horrendous style but it is entirely possible. In this example the data referenced by the xvalue is not immediately trashed after use, and it is bad to use terminlogy which suggests the opposite.
   
An xvalue refers to data that is flagged as movable. Not all xvalues are guaranteed to be temporary. We are better off defining things for what they actually are instead of how we expect them to be used.

### Value categories in ugly detail

What follows is a precise categorization of all expressions told in a format extremely similar to cppreference's page about value categories. This is not meant to be read front-to-back; I would recommend using it as a reference. Please keep in mind that operator overloading can completely override anything describe here, in which case the implementation of the override (specifically its return value) will determine the value category of the overloaded operation. 

Common lvalues:
 - names of variables (or functions, templates, data members)
 - all assignment expressions (=, +=, *=, etc)
  - The variable that is assigned to is the result of the expression, hence the result of the expression is locatable. 
 - an expression using the "dereference" operator (*)
  - Data acquired from an explicit address in memory is obviously locatable. 
 - string literals
   - The specification demands that object referenced by a string persists through the lifetime of the program (the terminology they use is "static storage"), so it is safe to make string literals locatable.
 - _Key properties:_
   - `&` is defined (please see an exception to this case in the discussion regarding functions below)
   - can be used as left operand of assignment operators, but only if value is modifiable

Common prvalues:
 - any non-string literal (`7`, `3.3f`, etc)
 - arithmetic expressions (`+`, `-`, `*`, `%`, etc)
 - logical expressions (`&&`, `||`, etc)
 - comparison expressions (`<`, `>`, `>=`, etc)
 - enumerator
   - enumerators are evaluated at compile time. the resultant machine code is indistinguishable from C++ code which uses an integer literal instead of an enum.
 - lambda expression
 -  _Key properties:_
    - `&` throws
    - cannot be used in left operand of any assignment
    - can be used to initialize const lvalue reference (`const MyClass& myRef = <prvalue>;`)
    - can be used to initialize rvalue reference (`MyClass&& myRef = <prvalue>;`)
    - function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function

Common xvalues:
 - a cast expression to rvalue reference to object type (static_cast<MyClass&&>(myClassInstance))
 - or a function that returns an rvalue reference to object type (std::move(myClassInstance))
 - _Key properties:_
   - `&` throws
   - cannot be used in left operand of any assignment
   - can be used to initialize const non-rvalue reference (`const MyClass& myRef = <xvalue>;`)
   - can be used to initialize rvalue reference (`MyClass&& myRef = <xvalue>;`)
   - function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function

### Additional cases: 

function names:
 - A function name that is not a class method is always an lvalue. However, if a function is overloaded, giving the function name to `&` is ambiguous so the compiler will throw. As far as I am aware this is the only case where an lvalue cannot be given to the `&` operator.  

function calls:
 - if the return type is not a reference, then prvalue.
     - Function calls almost always compute prvalues.
 - if the return type is an lvalue reference, then the function call is an lvalue
     - (This is only useful to create operator overloads which return lvalues. Any other usage is a severe code smell.)
 - if the return type is an rvalue reference of object type, then the function call is an xvalue
     - (This seems to exists mostly so that `std::move` could be added to the language. I haven't run into another situation where this is useful.)

comma operator
 - the value category of the final comma-separated expression is the value category of the comma expression

ternary
 - the value category of the resulting branch is the value category of the ternary expression

address-of operator (`&`)
 - a prvalue.
 - one might argue that this represents identity since the value of an address-of expression is the literal address of some object. However, it is possible to get the address of something that is later deallocated and as such we cannot guarantee forever that the result of a & operator is always the identity of an object. Contrast this with a reference which is guaranteed to point to some value.

subscript (`a[n]`)
 - an lvalue if one operand is an lvalue, an xvalue if one operand is an rvalue
  - Since a subscript is fully equivalent to a dereference (*(a + n)) an array lookup is an evaluation which guarantees the identity of something. But it is unsafe to make this result locatable if `a` is not locatable (if `a` is an rvalue), so `a[n]` it must be some type of rvalue if `a` is an rvalue. The only rvalue which carries identity is the xvalue so we see this must be an xvalue if `a` is an rvalue.

pre-increment, pre-decrement (`++a`, `--a`)
 - an lvalue (it is addressable since the result of the expression is the new value of the variable)

post-increment, post-decrement (a++, a--)
 - a prvalue (it is temporary since the result of the expression is different from the value of the variable in subsequent lines of code)

`a.m`
 -non-function, non-enumerator member
  - an lvalue if a is an lvalue, an xvalue if a is an rvalue
  ```
  class A {
    public:
      int b;
  };

  A foo() {
    return A();
  }

  int main() {
    A a;
    
    a.b;     // lvalue
    foo().b  // xvalue
  }
  ```
   - Classes are a generalization of C structs, and one non-enumerator data member of a struct can be accessed from another using pointer arithmetic (yes, this means there are situations where you can access private fields of an objects. Very unfortunate language design.) Hence data member access should be treated in the same manner as array access. See my disucssion about the subscript operator for more information.


 -non-function, enumerator member
  - prvalue
  ```
  class A {
    public:
      enum {
        b = 1
      };
  };

  int main() {
    A a;
    
    a.b;     // prvalue
  }
  ```

    - enumerators are evaluated at compile time. as such there is no "enum object" created at runtime whose address is looked up when an enum is used in your program. the value of that enum is hard-coded into the machine instructions for your program, and as such the resulting program is indistinguishable from one in which you used a integer literal (if the enum is of integer type) instead of the enum. this is why enums are not lvalues.

`p->m`
 -non-function, non-enumerator member
  - an lvalue

 -non-function, enumerator member
  - a prvalue

`a.*p`, `a->*p`
 - an lvalue if a is an lvalue, an xvalue if a is an rvalue

function member access of object (`a.f`, `p->f`, `a.*pf`, `p->*pf`)
 - a prvalue. these are particularly restricted, they can only be used as the left-hand argument of a function call and the compiler will throw if you try to use this expression for anything other than a function call. (you could argue this implies the existence of an additional value category if you wanted to make C++ developers lives even harder).
 - I do not know why these are prvalues If 

this
 - prvalue
 - `this` is a pointer. See my discussion about `&`; the same argument can be used to argue we do not consider pointers to have identity. 

template parameters
 - non-type template parameter of lvalue reference type
  - lvalue
 - non-type template parameter of a scalar type
  - prvalue

requires expression
 - prvalue

specialization of a concept
 - prvalue

cast expressions
 - xvalue if casted to rvalue reference of object type (by definition)
 - lvalue if casted to lvalue reference type
   - (I really don't know why lvalue reference casts are not xvalues, for all the same reasoning that rvalue reference casts of objects are xvalues. However, the C++98 standard decided these were lvalues, and no changes to the taxonomy from C++11 was going to change something that once was an lvalue into an rvalue, so this is easy to explain from C++'s insistance on backwards-compatibility. Furthermore, move semantics would become even more confusing than they already are if both lvalue reference casts and rvalue reference casts could be moved from.)
 - prvalue otherwise

void expressions
 - void functions, casting something as void, and throw expressions are all prvalues, but these expressions are not allowed to be used as function arguments or used to initialize references. throw expressions may be used in either branch of the ternary operator. (you could argue this implies the existence of an additional value category if you wanted to make C++ developers lives even harder).



### Properties of value categories: 

lvalue
 > & is defined
 > can be used as left operand of assignment operators (if value is modifiable)
 > can be used to initialize a non-rvalue reference
 > can be converted to prvalue with implicit conversion
   > lvalue-to-rvalue
   > array-to-pointer
   > function-to-pointer
 > can be polymorphic
 > can have incomplete type

prvalue
  > & throws
  > cannot be used in left operand of any assignment
  > can be used to initialize const lvalue reference
  > can be used to initialize rvalue reference
  > function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function
  > cannot be polymorphic
  > cannot have incomplete type (except void, or when used in decltype specifier)
  > cannot have abstract class type or an array of abstract class type

xvalue
  > & throws
  > cannot be used in left operand of any assignment
  > can be used to initialize const non-rvalue reference
  > can be used to initialize rvalue reference
  > function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function
  > can be converted to prvalue with implicit conversion
    > lvalue-to-rvalue
    > array-to-pointer
    > function-to-pointer
  > can be polymorphic
  > can have incomplete type

glvalue (ie, properties shared between lvalues and xvalues)
  > can be converted to prvalue with implicit conversion
    > lvalue-to-rvalue
    > array-to-pointer
    > function-to-pointer
  > can be polymorphic
  > can have incomplete type

rvalue (ie, properties shared between prvalues and xvalues)
  > & throws
  > cannot be used in left operand of any assignment
  > can be used to initialize const non-rvalue reference
  > can be used to initialize rvalue reference
  > function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function
