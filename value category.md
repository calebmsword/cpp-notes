Any expression in C++ can fall into one of a small number of categories called **value categories** of which the two most important are **rvalues** and **lvalues**. All expressions are either an rvalue or lvalue. This terminology is often criticized, and rightly so. First of all, they should be called "expression categories", as they categorize expressions, not values, and we should have "l-expression" and "r-expression" categories instead of rvalue and lvalue. Furthermore, lvalue and rvalue brings to mind "left value" and "right values", and you might think there may be some significant correlation between leftness/rightness and lvalue/rvalue. The only meaningful correlation is that lvalues and rvalues are opposites; an expression cannot be both an rvalue and an lvalue at the same time. Trying to find any further correlations only leads to confusion.

<details>
<summary>Note</summary>
<br>

The names "lvalue" and "rvalue" are historical artifacts. The esotric language [CPL](http://www.math.bas.bg/~bantchev/place/cpl/features.pdf) (The Combined Programming Language) allowed expressions to be evaluated in "left-hand" mode, where the expression is on the left side of a value assignment, and any expression not evaluated in left-hand mode was evaluated in "right-hand" mode. C inherited this terminology with the concept of the lvalue. According to section A5 from the "The C Programming Language", 2nd Edition from Kernighan and Ritchie,

> An <i>object</i> is a named region of storage; and <i>lvalue</i> is an expression referring to an object. An obvious example of an lvalue expression is an identifier with suitable type and storage class. There are operators that yield lavalues: for example, if `E` is an expression of pointer type, then `*E` is an lvalue expression referring to the object to which `E` points. The name "lvalue" comes from the assignment expression `E1 = E2` in which the left operation `E1` must be an lvalue expression.

Since C++ is (almost) a superset of C, it also inherits the concept of an lvalue and also introduces the term "rvalue" for anything that isn't an lvalue, but C++'s additional complexity results in situations where an lvalue cannot appear on the left hand side of an assignment or where an rvalue appears on the left hand side of an assignment. To see why I discourage you from thinking of lvalues and rvalues as what can be allowed to appear on either side of an `=` operation, consider the following snippet:

```c++
#include <iostream>

class A{};

int main() {
  const A a;
  // a = A();  // this would throw; a is const and cannot be modified
  std::cout << &(A() = a) << "\n";  // the result of a constructor is an rvalue, yet this is allowed to compile
}
```

Operator overloads can completely hijack the expected behavior of operators, and trivially-copiable classes are implicitly compiled with an operator overload that allows rvalues to appear on the right hand side of a copy assignment.

<br>
<br>
</details>

It is easiest to define value categories by describing their historical evolution:

## Before C++11

Two defining characteristics of expressions are **value**, which is the actual value the expression computes, and an **address** which is the physical location in memory where the value is stored. (Expressions in C++ also have **types** but that is not signficant for this discussion.)  Despite what the name suggests, a "value category" tells us something about the __address__ of any given expression:

 - lvalues were expressions whose addresses are guaranteed to made available to the programmer. Hence, we will say that lvalues are **locatable**.
 - There is no guarantee the address of an rvalues will be made available to the programmer. Hence, we will say that rvalues are **latent**.

Most of the time, there is a clear reason the language makes certain values locatable and certain values latent. Some examples will elucidate:

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
It is a common misconception that lvalues and rvalues indicate the lifetime of data; one might think that lvalues are expressions whose data has persistent lifetime and rvalues are expressions whose data has temporary lifetime. The third example shows why rvalues are not, in general, temporary. I regret to inform that some abuses of the language specification make it possible to create lvalues that refer to data that is no longer in scope. The lesson is clear: in general, do not associate value category with lifetime.
</details>

## C++11

The changes to value category taxonomy was a result of C++11's introduction of **move semantics**. While not necessary to know for the discussion here, it still might be helpful to have that context. Feel free to open the following dropdown to see a brief overview of move semantics. 

<details>
<summary>C++ move semantics</summary>

<br>

We use the term **resource** to describe something that must be freed in an exception-safe way. Resources are things like allocated memory, an open file, a lock, a connection to a socket, etc. (If you have used a try-catch-finally block (Java), a `with` statement (Python), or a defer statements (Go), then you were almost certainly using a "resource".) The simplest way to guarantee in C++ that a resource is safely acquired is to create classes whose instantiated objects manage a single resource such that the constructor acquires the resource and the destructor frees the resource. Since a destructor is called whenever an object goes out of scope, a destructor is guaranteed to be called even if an exception occurs. (This design pattern is so ubiquitous in the C++ community that is has the name RAII, short for Resource Acquisition Is Initialization.)

It can be useful to exchange ownership of a resource from one object to another. This concept is called a **move**. If object B represents acquired memory allocation and wants to exchange ownership of this allocated memory to object A, the allocated memory is represented as a field whose value is a pointer to that memory. The move will assign this pointer to the appropriate field A and make sure B now has a `nullptr` in this field, indicating that it no longer owns a resource.

This concept is useful enough that the C++ standards committee wanted to add language semantics which allow for a stylistic standard for creating moves. They chose to implement this by adding a new type of reference. Constructors and assignment overloads could be written by the programmer which accept this new type of reference, by which it is understood by the programmer that references of this type given to a constructor or assignment are intended to be moved from. This new reference is referred to as an `rvalue reference`, and the pre-C++11 reference was renamed to an `lvalue reference`. As the name suggests, an rvalue reference can only be bound to rvalues. Significantly, if you overloaded a function such that one signature takes an lvalue reference and the other takes an rvalue reference, you to create a function which has different behavior depending on whether the argument provided is an rvalue or lvalue. The rvalue reference is also a mutable reference if declared non-const. Before C++11, rvalues could only be bound to `const` lvalue references meaning there was no way to mutate a reference bound to an rvalue.

The C++ standards committee added a function `std::move` to the library which takes a single argument and returns an rvalue reference to the object given to the function. (This function should have been called `std::as_movable` since no move occurs in the execution of `std::move`, a common point of confusion among new C++ developers.) If the user provides the result of `std::move` to a constructor or assignment which is overloaded to accept an rvalue reference, we cause a move if these overloads exist and are implemented in the expected manner. A constructor of a class `T` which accepts a `T` rvalue reference parameter is called a **move constructor**. An assignment operator overload of class `T` which accepts a `T` rvalue reference parameter is called a **move assignment overload**. Evidently, we should interpret the return value of `std::move` as a reference which acts as a flag that the object can be moved from.

The C++ standard uses the term **movable** to describe objects which induce an rvalue reference function signature to be executed during overload resolution. This means "movable" and "rvalue" are synonymous.

<br>
<br>

</details>

In C++11, the standards committee wanted to categorize expressions whose evaluation determines an object's **identity**. Expressions determine identity if their value is determined by calculating a specific address in memory, act as an umabigious identifier for some specific object, or if they are lvalues. There are many types of expressions which are treated as "determining identity":

 1) **lvalues**. This is because the address of an object is immediately available to the programmer if they use the `&` operator of any lvalue, and we can consider the address of an object as its identity.
 2) **Expressions that evaluate to references**. A reference is an alias to an existing object, cannot be reassigned, and if necessary extends the lifetime of the referenced object to that of the reference. We have extremely high assurance that a reference refers to one specific object and therefore the reference is treated as an identity. There are two types of expressions which evaluate a reference: 1) a function call of a function which returns a reference, or 2) a cast to a reference.
 3) **Array subscript operations**. When `a` is an array then `a[n]` is syntactic sugar for `*(a + n)`. We are explicitly evaluating an address in memory and accessing whatever object is there, and as such this evaluation is treated as "determining identity".
 4) **Non-enumerator, non-static-function data member access**. Enumerators are evaluated at compile time so is no address associated with an enumerator at runtime, and as such enumerators are not treated as something with identity. Function members are sometimes evaluated at runtime due to virtual functions, so it is sometimes ambigious as to what function is being referred to by a function member access, and as such the language does not consider these to carry identity either. In all other cases, member access acts a name to a specific object in memory and as such the standard treats the operation as one that "determines identity".
 5) **Static member function access** is functionally equivalent to accessing a function within a namespace. Since ordinary function names are treated as lvalues, the language also treats member access of static functions as lvalues.

The C++ standards committee recognized the existence of latent expressions with identity when they introduced the *rvalue reference cast* introduced in C++11. An rvalue reference of object type results in latent data (the language specification does not allow this expression to be provided to `&`) but since it is a reference its value *is* the identity of that object. The C++ standards committee felt this expression was best represented by a third value category and the pre-existing value category taxonomy was salvaged by changing the meaning of the word "rvalue":
 - nearly all of what we used to call rvalues are now called **prvalues** ("pure rvalues"),
 - what we used to call lvalues are still called lvalues,
 - an __rvalue reference cast expression of an object__ is of a new value category called **xvalue**,
   - __a function which returns an rvalue reference of an object__ is also considered an xvalue,
 - and rvalue is now an umbrella term: __xvalues and prvalues are specific types of rvalues__.

<details>
<summary>Note</summary>
<br>
There also exists the less useful umbrella term <b>glvalue</b>, short for "general lvalue". xvalues and lvalues are specific types of glvalues. An xvalue is both a glvalue and an rvalue.
<br>
<br>
</details>

<details>
<summary>Note</summary>
<br>
The only way to create an rvalue reference cast expression of non-object type is if you cast a function to an rvalue reference. rvalue references of functions are treated as lvalues. Most situations where a reference to a function could be used are best solved with lambdas so this is not a commonly used language feature.
<br>
<br>
</details>

As a result of this change, everything that was an lvalue before was still an lvalue and everything that used to be an rvalue was still an rvalue, but now there two specific and mutually-exclusive types of rvalues. The xvalue is a latent object with identity. prvalues are latent objects without identity.

<details>
<summary>Note</summary>
<br>
The C++11 standard defines rvalues as expressions which are movable. This is a circular definition; see the overview of move semantics at the beginning of this section. I believe is better to continue to define rvalues as latent expressions even after the C++11 standard.
<br>
<br>
</details>

The term "xvalue" was originally introduced without any meaning. I prefer to think of xvalue as being short for "cross value" since an xvalue contains a cross of a characteristic usually associated with lvalues (identity) and the characteristic of pre-C++11 rvalues (latency). There is a small class of expressions that were rvalues before C++11 that are now also xvalues, but the rvalue cast expression and the function which returns an rvalue reference are by far the most important and common examples encountered in practice.

<details>
<summary>Note</summary>
<br>
<p>Things that were rvalues before C++11 that the standards committee recognizes as xvalues are:</p>
<ol>
  <li><b>Non-static, non-enumerator, non-function member access of an rvalue</b>. Static data members are still treated as locatable since static members are not associated with the lifetime of a class/struct instance.</li>
  <li><b>Array subscripting of an rvalue array</b>.</li>
</ol>
 
<br>
<br>
</details>

## C++17

C++17 added more wrinkles to the taxonomy. Before C++17, we could have a prvalue which represented a latent object (for example, a function call of a function whose return statement calls a class constructor). After C++17, it no longer represents an object but instead acts as a "free coupon" for that result object. Hence there is no expensive object copy when a prvalue result is returned by a function, we simply "xerox the coupon" for the actual resultant object. The specification demands that, at some point, the object is **materialized** (the coupon is exchanged) into the actual result object, and which point the specification also demands that the prvalue is converted into an xvalue. We will use the term **immaterial** to describe a prvalue which eventually materializes into an xvalue. 

Most prvalues are now immaterials that are eventually materialized into an xvalue. This allows the language to remove unnecessary copies of objects. With this new feature, we can no longer bind prvalues to rvalue references. Instead, the prvalue is implicitly materialized into an xvalue and that xvalue is bound to the reference. (Before C++17 it was said that all rvalues are movable. Now, only xvalues are movable.)

There are situations where a prvalue is not an immaterial. For example, if we assign a variable to some lvalue expression:

```c++
int() {
  int b = 7;
  int a = b;  // lvalue "b" is assigned to "a"
}
```

The language implicitly converts the lvalue to an prvalue in a process known as **lvalue-to-rvalue** conversion. (This is not behavior new to C++17; the language has always behaved in this way.) It should be understood that there are [many other situations](https://en.cppreference.com/w/cpp/language/implicit_conversion#Lvalue-to-rvalue_conversion) where lvalue-to-rvalue conversions occur. The vast majority of lvalue-to-rvalue conversions do not create immaterial prvalues--they behave exactly like C++11 prvalues.  

## Summary

In short:
 - the distinguishing factor between lvalues and rvalues is whether or not the address of the value of the expression is guaranteed to be available to the programmer
 - lvalues are **locatable**--their address is guaranteed to be made available to the programmer
 - rvalues are **latent**--their address is only made available through assignment to a reference which allows indirect access to the address of the value
 - changes to the C++11 and C++17 do not change these facts, and only serve to introduce two specific types of rvalues (prvalues and xvalues)
 - after C++11,
   - xvalues are latent expressions with identity, and
   - prvalues are latent expressions without identity 
 - after C++17.
   - prvalues not created from lvalue-to-rvalue conversions are immaterial representations of result objects, and
   - xvalues can now also represent materializations of immaterial prvalues.

### An aside on the xvalue terminology

I will mention that the standard says that xvalues are "eXpiring values", terminology I strongly dislike. Since most practical usage of xvalues is to call a move constructor or move assignment, the value represented by the xvalue is meant to be "moved from" and eventually discarded, which I suppose what this terminology is supposed to mean. This hinges on the programmer adhering to convention, however:

 - The class type might not have a move constructor or move assignment overload. Then the xvalue won't be moved from at all.
 - The move constructor or move assignment overload could be implemented in a faulty or nefarious way which does not perform the expected behavior.
 - The user could have an lvalue pointing to some object, cast it as an xvalue and move it to some other variable, and then access the original lvalue and manipulate the object that was moved from. Such a thing would be considered horrendous style but it is entirely possible. In this example the data referenced by the xvalue is not immediately trashed after use, and it is bad to use terminlogy which suggests the opposite.
   
An xvalue is either 1) a latent expression with identity or 2) a materialization of a prvalue. There is no guarantee that an xvalue is expiring/temporary. We are better off defining things for what they actually are instead of how we expect them to be used.

### Value categories in ugly detail

What follows is a precise categorization of all expressions told in a format extremely similar to cppreference's page about value categories. This is not meant to be read front-to-back; I would recommend using it as a reference. Please keep in mind that operator overloading can completely override anything describe here, in which case the implementation of the override (specifically its return value, see the discussion below on the value categories of function calls) will determine the value category of the overloaded operation. 

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
 - a cast expression to rvalue reference to object type (`static_cast<MyClass&&>(myClassInstance)`)
 - or a function that returns an rvalue reference to object type (`std::move(myClassInstance)`)
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
 - one might argue that this represents identity since the value of an address-of expression is the literal address of some object. However, it is possible to get the address of something that is later deallocated and as such we cannot guarantee forever that the result of a & operator is always the identity of an object. Contrast this with a reference, which we can have high confidence actually points to something. (It is, unfortunately,  possible to cause references to dangle with abuses of the language specification. Try, for example, `int& null_reference = *static_cast<int*>(nullptr);`.)

subscript (`a[n]`)
 - an lvalue if one operand is an lvalue, an xvalue if one operand is an rvalue
  - Since a subscript is fully equivalent to a dereference (*(a + n)) an array lookup is an evaluation which guarantees the identity of something. But it is unsafe to make this result locatable if `a` is not locatable (if `a` is an rvalue), so `a[n]` it must be some type of rvalue if `a` is an rvalue. The only rvalue which carries identity is the xvalue so we see this must be an xvalue if `a` is an rvalue.

pre-increment, pre-decrement (`++a`, `--a`)
 - an lvalue (it is addressable since the result of the expression is the new value of the variable)

post-increment, post-decrement (`a++`, `a--`)
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
`
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
 - If the method is declared virtual and you access that method through a pointer or reference, the actual method invoked is determined at runtime. The compiler does not actually know the specific address of the method in that case. Why a non-pointer/non-reference member access is also treated as a prvalue is a mystery to me as their is no ambiguity to the function that will be called.

this
 - prvalue
 - This is perplexing to me, I do not understand this choice from the language specification. I will update these notes if I ever find a satisfying justification for this decision.

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
   - (It feels arbitrary that an lvalue reference cast is locatable while an rvalue reference cast is latent. However, if we had an overloaded function with a lvalue reference parameter and an rvalue reference parameter, it would be immensely confusing if an lvalue reference cast caused an rvalue reference overload to be called.)
 - prvalue otherwise

void expressions
 - void functions, casting something as void, and throw expressions are all prvalues, but these expressions are not allowed to be used as function arguments or used to initialize references. throw expressions may be used in either branch of the ternary operator. (you could argue this implies the existence of an additional value category if you wanted to make C++ developers lives even harder).

### Properties of value categories: 

lvalue
 - & is defined
 - can be used as left operand of assignment operators (if value is modifiable)
 - can be used to initialize a non-rvalue reference
 - can be converted to prvalue with implicit conversion
   - lvalue-to-rvalue
   - array-to-pointer
   - function-to-pointer
 - can be polymorphic
 - can have incomplete type

prvalue
  - & throws
  - cannot be used in left operand of any assignment
  - can be used to initialize const lvalue reference
  - can be used to initialize rvalue reference
  - function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function
  - cannot be polymorphic
  - cannot have incomplete type (except void, or when used in decltype specifier)
  - cannot have abstract class type or an array of abstract class type

xvalue
  - & throws
  - cannot be used in left operand of any assignment
  - can be used to initialize const non-rvalue reference
  - can be used to initialize rvalue reference
  - function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function
  - can be converted to prvalue with implicit conversion
    - lvalue-to-rvalue
    - array-to-pointer
    - function-to-pointer
  - can be polymorphic
  - can have incomplete type

glvalue (ie, properties shared between lvalues and xvalues)
  - can be converted to prvalue with implicit conversion
    - lvalue-to-rvalue
    - array-to-pointer
    - function-to-pointer
  - can be polymorphic
  - can have incomplete type

rvalue (ie, properties shared between prvalues and xvalues)
  - & throws
  - cannot be used in left operand of any assignment
  - can be used to initialize const non-rvalue reference
  - can be used to initialize rvalue reference
  - function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function

# Acknowledgements

 - cppreference's [page](https://en.cppreference.com/w/cpp/language/value_category) on value categories was invaluable for this writeup.
 - The [official drafts](https://www.open-std.org/JTC1/SC22/WG21/docs/standards) for the C++ specification were also an invaluable resource.
 - Arno Schödl's [talk](https://www.youtube.com/watch?v=s9vBk5CxFyY) "The C++ rvalue lifetime disaster" didn't specifically influence anything specific on this writeup but the time I spent afterwards ruminating on the lifetimes of rvalues was invaluable.
 - Arthur O'Dwyer's blog posts ([this](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/) and [this](https://quuxplusone.github.io/blog/2020/03/04/rvalue-lifetime-disaster/)) regarding C++ lifetime encouraged me to avoid using the term "temporary" when discussing rvalues. I would not have introduced the concept of "rvalues are latent" without these writeups.
 - Anders Schau Knatten's [talk](https://www.youtube.com/watch?v=hkyZ8L343cU) "lvalues, rvalues, glvalues, prvalues, xvalues, help!", while a talk I dislike overall, introduced the shocking edge case that a `std::string` can appear on the left hand side of an assignment operation which helped me learn about default copy assignment overload behavior.
 - "The C Programming Language" 2nd Edition by Kernighan and Ritchie, which I used to provide an "official" description of lvalues in C.
 - "The Main Features of CPL" by D. W. Barron, et al was helpful for discussing the history of the concept of an lvalue.
