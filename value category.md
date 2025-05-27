Any expression in C++ can fall into one of a small number of categories called **value categories** of which the two most important are **rvalues** and **lvalues**. All expressions are either an rvalue or lvalue. This terminology suggests that there may be some significant correlation between leftness/rightness and lvalue/rvalue. The only meaningful correlation is that lvalues and rvalues are opposites; an expression cannot be both an rvalue and an lvalue at the same time. Trying to find any further correlations only leads to confusion.

<details>
<summary>Note</summary>
<br>

Historically, the term lvalue has been used to refer to expressions which can appear on the left hand side of an assignment. The esoteric language [CPL](http://www.math.bas.bg/~bantchev/place/cpl/features.pdf) (The Combined Programming Language) allowed expressions to be evaluated in "left-hand" mode, where the expression is on the left side of a value assignment, and any expression not evaluated in left-hand mode was evaluated in "right-hand" mode. C, which was influenced by CPL, inherited this concept and also called it an "lvalue". According to section A5 from the "The C Programming Language", 2nd Edition from Kernighan and Ritchie,

> An <i>object</i> is a named region of storage; and <i>lvalue</i> is an expression referring to an object. An obvious example of an lvalue expression is an identifier with suitable type and storage class. There are operators that yield lvalues: for example, if `E` is an expression of pointer type, then `*E` is an lvalue expression referring to the object to which `E` points. The name "lvalue" comes from the assignment expression `E1 = E2` in which the left operation `E1` must be an lvalue expression.

Since C++ is (almost) a superset of C, it also inherits the concept of an lvalue, and also introduces the term "rvalue" for anything that isn't an lvalue. However, while lvalue historically refers to expressions that can appear on the left side of an assignment, C++'s additional complexity results in situations where an lvalue cannot appear on the left hand side of an assignment and operator overloading can allow an rvalue to appear on the left hand side of an assignment. Consider the following snippet:

```c++
#include <iostream>

class A{};

int main() {
  const A a;
  // a = A();  // this would throw; a is const and cannot be modified
  std::cout << &(A() = a) << "\n";  // the result of a constructor is an rvalue, yet this is allowed to compile
}
```

(Trivially-copiable classes are implicitly compiled with an operator overload that allows rvalues to appear on the left hand side of a copy assignment.)

In C++ we should not think of lvalues as things that must appear on the left-hand side of an assignment; it only leads to confusion.

<br>
<br>
</details>

It is easiest to define value categories by describing their historical evolution:

## Before C++11

Two defining characteristics of expressions are **value**, which is the actual value the expression computes, and an **address** which is the physical location in memory where the value is stored. (Expressions in C++ also have **types** but that is not signficant for this discussion.)  Despite what the name suggests, a "value category" tells us something about the __address__ of any given expression:

 - lvalues were expressions whose addresses are guaranteed to made available to the programmer through the `&` operator. Hence, we will say that lvalues are **locatable**.
 - The addresses of rvalues are not made available to the programmer through the `&` operator. Hence, we will say that rvalues are **latent**.

Most of the time, there is a clear reason the language makes certain expressions locatable and certain expressions latent. Some examples will elucidate:

```c++
int x = 3;
x; // this is an lvalue
```
 - the data stored in `x` is accessible to us with a named variable. Typically, names of things represent locatable data in C++. Since the data is stored in a variable, the line of code after the expression `x` still has access to the location in memory which stores `x`'s data. This is why `&` is defined for `x`, as it is safe to reveal this address to the programmer. That location in memory will have the data in `x` as long as `x` remains in scope.

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
 - Integer literals are, in general, rvalues. While rvalues are latent, there some situations where the address can be saved __if__ they stored into a reference. In this case, the location in memory for `3` will be referenced by `number_reference` and the lifetime of the value for the literal is extended such that it shares the lifetime of the reference. When we find the address of `number_reference` we find the address of the value that was represented by the expression `3`. That is, `3`'s address is made available indirectly through the reference.

<details>
<summary>Note</summary>
<br>
Some people call rvalues "temporary values", and unfortunately this sort of terminology has even permeated the C++ standard. But it is clear from the last example that rvalues are not guaranteed to "immediately vanish" by the next line of code. The terminology "temporary" is objectively incorrect so I am avoiding it entirely in this discussion.
</details>

<details>
<summary>Note</summary>
<br>
It is a common misconception that lvalues and rvalues indicate the lifetime of data; one might think that lvalues are expressions whose data has persistent lifetime and rvalues are expressions whose data has temporary lifetime. The third example shows why rvalues are not, in general, temporary. I regret to inform that some abuses of the language specification make it possible to create lvalues that refer to data that is no longer in scope. The lesson is clear: in general, do not associate value category with lifetime.
<br>
<br>
</details>

Here are some examples of the most common lvalues and rvalues: (it should be understood that these apply to default, built-in behaviors of C++ operators. Operator overloading can completely override any of the following facts.)

Common lvalues:
 - names of variables (or functions, templates, data members)
 - all assignment expressions (=, +=, *=, etc)
   - The variable that is assigned to is the result of the expression, hence the result of the expression is locatable. 
 - an expression using the "dereference" operator (*)
   - Data acquired from an explicit address in memory is obviously locatable. 
 - string literals
   - The specification demands that object referenced by a string persists through the lifetime of the program (the terminology they use is "static storage"), so it is safe to make string literals locatable.
 - _Key properties:_
   - `&` is defined (unless the lvalue is the name of an overloaded function, in which case the location is ambiguous)
   - can be used as left operand of assignment operators, but only if value is modifiable

Common rvalues:
 - any non-string literal (`7`, `3.3f`, etc)
 - arithmetic expressions (`+`, `-`, `*`, `%`, etc)
 - logical expressions (`&&`, `||`, etc)
 - comparison expressions (`<`, `>`, `>=`, etc)
 - function calls of any function that does not return an lvalue reference
 -  _Key properties:_
    - `&` throws
    - cannot be used in left operand of any assignment
    - can be used to initialize const lvalue reference (`const MyClass& myRef = <rvalue>;`)
    - can be used to initialize rvalue reference (`MyClass&& myRef = <rvalue>;`)
    - function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function

The definition of lvalues as locatables has one unfortunate edge case. Names of functions are considered lvalues, yet if you overload a function, passing that function to `&` results in a compilation failure since it is ambiguous which function you should receive the address of. Also note that static member functions of objects are are also treated as lvalues in C++ since a static function is functionally equivalent to a regular function defined in a namespace. 

Evidently, the definition of lvalues provided earlier is incomplete. A more complete definition is that lvalues are 1) locatables or 2) names of functions or static method member access.

<details>
<summary>Note</summary>
<br>
<p>It is worth examining the value categorization of methods:</p>

 - If we access the name of a method with an qualified-id (`MyClass::method_name`) the result is considered an lvalue.
 - Function member access of static functions (`my_instance.static_method`) the result is also considered an lvalue.
 - But non-static function member access is considered an *rvalue* (`my_instance.method_name`).

The names of functions are, in general, potentially ambiguous because overloading is always possible. Yet non-static member functions are the only function-like expression that is an rvalue. Perhaps the difference is that non-static function member access are sometimes *resolved at runtime* if the method is declared virtual. Whatever the reason, we are stuck with the decision from the C++ language designers that only functions and static methods are lvalues.
<br>
<br>
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

In C++11, the standards committee wanted to categorize expressions whose evaluation determines an object's **identity**. These expressions are said to "have identity". Expressions have identity if 1) their value is determined by calculating an address in memory, 2) they are an umabigious identifier for some object, or 3) they are lvalues. There are many types of expressions with identity:

1) <ins>expressions which calculate specific memory addresses</ins>:
   - **Array subscript operations**. When `a` is an array then `a[n]` is syntactic sugar for `*(a + n)`. We are explicitly evaluating an address in memory and accessing whatever object is there.
2) <ins>expressions that are umabigious identifiers for an object</ins>:
   - **Expressions that evaluate to references**. A reference is an alias to an existing object, cannot be reassigned, and if necessary extends the lifetime of the referenced object to that of the reference. We have extremely high assurance that a reference always refers to one specific object and therefore the reference is treated as an identity. There are two types of expressions which evaluate a reference: 1) a function call of a function which returns a reference, or 2) a cast to a reference.
   - **Non-enumerator, non-function data member access**. Enumerators are evaluated at compile time so is no address associated with an enumerator at runtime, and as such enumerators are not treated as something with identity. The existence of virtual functions means that it can be ambigious what function is being referred to by function member access, so the language does not consider these to carry identity either. In all other cases, member access functions a name to a specific object in memory and as such the standard treats the operation as one that "determines identity".
3) <ins>lvalues</ins>:
   - This is because the address of an object is immediately available to the programmer if they use the `&` operator of any lvalue, and we can consider the address of an object as its identity.
   - Unfortunately, this means that function names and static methods are said to have identity even though function names can be ambiguous (see the previous section).

What is signficant about this definition of identity is that <ins>it is possible for an rvalue to have identity</ins>. The C++ standards committee recognized the existence of latent expressions with identity when they introduced the *rvalue reference cast* introduced in C++11: an rvalue reference of object type results in latent data (the language specification does not allow this expression to be provided to `&`) but since it is a reference its value *is* the identity of that object. They soon noticed some other rvalues that have identity and felt they should be represented by a third value category. This new value category was integrated into the old value category system by changing the meaning of the word "rvalue":
 - nearly all of what we used to call rvalues are now called **prvalues** ("pure rvalues"),
 - what we used to call lvalues are still called lvalues,
 - latent objects with identity are now in a new category called **xvalue**,
 - and rvalue is now an umbrella term: __xvalues and prvalues are specific types of rvalues__.

There are only a few types of expressions that are xvalues after C++11:

  1) <ins>rvalue reference casts of objects</ins>.
  2) <ins>Functions that return rvalue references of objects</ins>.
  3) <ins>Non-static, non-enumerator, non-function member access of an rvalue</ins>. (Static data members are still treated as locatable since static members are not associated with the lifetime of a class/struct instance.)
  4) <ins>Array subscripting of an rvalue array</ins>.

```c++
#include <iostream>

class A {
public:
  int b = 3;
  static const int c = 4;
  enum {
    d = 0
  };
};

A&& get_A_ref() {
  return A(); 
}

static int arr[10];

int* get_array() {
  return arr;
}

int main() {
  A a;
  
  A&& aref1 = static_cast<A&&>(a);      // xvalue of type 1)
  A&& aref2 = get_A_ref();              // xvalue of type 2)
  std::cout << A().b << "\n";           // xvalue of type 3)
  std::cout << A().c << "\n";           // NOT an xvalue, static fields are lvalues!
  std::cout << A().d << "\n";           // NOT an xvalue, enumerator fields are prvalues!
  std::cout << get_array()[0] << "\n";  // xvalue of type 4)
}
```

xvalues of type 2) are the most common type of xvalue. This is because of the standard library function `std::move`, a function which simply returns an rvalue reference bound to its argument. `std::move` is almost always used to perform moves.

<details>
<summary>Note</summary>
<br>
The only way to create an rvalue reference cast expression of non-object type is if you cast a function to an rvalue reference. rvalue references of functions are treated as lvalues. (Most situations where a function reference is appropriate are better solved with lambdas so this is not a commonly-used language feature.)
<br>
<br>
</details>

<details>
<summary>Note</summary>
<br>
There also exists the umbrella term <b>glvalue</b>, short for "general lvalue", which categories all expressions with identity. xvalues and lvalues are specific types of glvalues. An xvalue is both a glvalue and an rvalue.

This term is not useful for everyday C++ programming but it appears frequently in the C++ standard.
<br>
<br>
</details>

The result of the C++11 revamp of value categories is that everything that was an lvalue before was still an lvalue and everything that used to be an rvalue was still an rvalue, but now there two specific and mutually-exclusive types of rvalues. The xvalue is a latent object with identity. prvalues are latent objects without identity.

You might wonder what the term xvalue is supposed to mean. I prefer to think of xvalue as being short for "cross value" since an xvalue contains a cross of a feature usually associated with lvalues (identity) and the feature of rvalues (they are latent).

<details>
<summary>Note</summary>
<br>
<p>The term "xvalue" was understood by the C++ standards committee to be a vague and arbitrary name. To quote Bjarne Stroustrup's <a href="https://www.stroustrup.com/terminology.pdf">2013 document</a> on the xvalue terminology:</p>

> "We really don’t have anything that guides us to a good name for those esoteric beasts. They are important to people working with the (draft) standard text, but are unlikely to become a household name. We didn’t find any real constraints on the naming to guide us, so we picked ‘x’ for the center, the unknown, the strange, the xpert only, or even x-rated."

<br>
<br>
</details>

<details>
<summary>Note</summary>
<br>
The C++11 standard defines rvalues as expressions which are movable. This is a circular definition; see the overview of move semantics at the beginning of this section. I believe is better to continue to define rvalues as latent expressions even after the C++11 standard.
<br>
<br>
</details>

## C++17

More changes to value categorization was introduced due to the standardization of **copy elision** in C++17.

The fact that objects are passed-by-value into functions results in potentially-expensive copy operations whenever functions are used with non-reference/non-pointer arguments. However, some compilers would perform "copy elisions" in some situations to avoid unnecessary copies. A simple example is the **U**nnamed **R**eturn **V**alue **O**ptimization (URVO):

```c++
class T {};

T get_T() {
  return T();
}

int main() {
  T t = get_T();
}
```

In this code example, the pre-C++17 standard demands that a `T` object is copied twice (the return value of `get_T` is copied into the scope of `main` and then the copy assignment induces a second copy). However, with URVO, only one `T` object is initialized and it is stored directly into `t`.

Such copy elisions were commonplace enough in C++ compilers that the standards committee felt it was safest for the language to guarantee copy elision in some places to enforce consistency among language implementations. As such, the C++17 specification guarantees "copy elision" in a small number of situations. cppreference only describes two situations where copies are elided:

 - when a return statement is an initialization of an object of the same type as the function return type
 - when an object initialization assignment has an rvalue on the right side of the equals sign

(URVO as shown above uses both types of copy elision.) The C++17 specification implements these "guaranteed copy elisions" by changing the meaning of prvalues and xvalues.

After C++17, a prvalue now represents the promise of an eventual object initialization. The initialization of the prvalue is called a **materialization** in which case the prvalue is implicitly converted into an xvalue. With this change, prvalues are now lightweight representations of some eventual result. The initialization of the result is delayed as long as possible in order to elide unnecessary copies. I like use the analogy of a "prvalue as a free coupon". A coupon is representation of some item that can be eventually exchanged. In this analogy, returning a prvalue from a function is simply "xeroxing the coupon" between scopes instead of cloning the expensive item it represents. Eventually, the prvalue is materialized, which in this analogy is represented by exchanging the free coupon for its promised item.

The C++ standard does not provide a convenient terminology for these new prvalues. I use the term **immaterial** to describe prvalues whose results are eventually **materialized** into xvalues. It is also important to understand that everything that was an xvalue in C++11 is still an xvalue, and that C++17 now allows xvalues to categorize an additional type of expression (a materialization of a prvalue).

There is one more edge case. In C++17, rvalues of void type are considered prvalues. void types have no concept of initialization, so we should think of this as a special type of prvalue that has no concept of materialization.

As such, in C++17:
 - prvalues are now 1) immaterial representations of a result, or 2) latents of type void
 - xvalues are either 1) latents with identity, or 2) materializations of prvalues

<details>

<summary>Note</summary>

The official C++17 standard defines a prvalue as

 > an expression whose evaluation initializes an object or a bit-field, or computes the value of the operand of an operator, as specified by the context in which it appears.

To reconcile this definition with the previous discussion, realize that prvalues that are not yet materialized are said to "[compute] the value of the operand of an operator, as specified by the context in which it appears." The specification later breaks down each operator in excruciating detail and specifically describes how prvalues are handled for each operator that takes a prvalue for its operand(s)--this is the meaning of the phrase "as specified by the context in which it appears." Meanwhile, prvalues which are materialized are said to "[initialize] an object or a bit-field".

It may also be useful to be aware of the term "result object". The specification describes it as such:

 > The result object of a prvalue is the object initialized by the prvalue; a prvalue that is used to compute the value of an operand of an operator or that has type cv void has no result object.

</details>

## Summary

In short:
 - lvalues are either
   - **locatables**: expressions whose address is guaranteed to be made immediately available through the `&` operator
   - names of functions or static method member access
 - rvalues are **latent**: their address is not immediately available; they cannot be provided to the `&` operator
 - changes to C++ from the C++11 and C++17 standards do not change these facts, and instead introduce two specific types of rvalues (prvalues and xvalues)
 - after C++11,
   - all rvalues are either an xvalue or prvalue,
   - xvalues are latent expressions with identity, and
   - prvalues are latent expressions without identity.
 - after C++17,
   - prvalues are 1) immaterial representations of a result, or 2) latents of type void
   - xvalues are either 1) latent expressions with identity, or 2) materializations of immaterial prvalues.

The terms **locatable**, **latent**, and **immaterial** are non-standard terminologies I invented for this write-up. Do not expect other developers to know what they mean. (But please feel free to spread their usage.)

## An aside on the xvalue terminology

I will mention that the standard says that xvalues are "eXpiring values", terminology I strongly dislike. Since most practical usage of xvalues is to call a move constructor or move assignment, the value represented by the xvalue is meant to be "moved from" and eventually discarded, which I suppose what this terminology is supposed to mean. This hinges on the programmer adhering to convention, however:

 - The class type might not have a move constructor or move assignment overload. Then the xvalue won't be moved from at all.
 - The move constructor or move assignment overload could be implemented in a faulty or nefarious way which does not perform the expected behavior.
 - The user could have an lvalue pointing to some object, cast it as an xvalue and move it to some other variable, and then access the original lvalue and manipulate the object that was moved from. Such a thing would be considered horrendous style but it is entirely possible. In this example the data referenced by the xvalue is not immediately trashed after use, and it is bad to use terminlogy which suggests the opposite.
   
An xvalue is either 1) a latent expression with identity or 2) a materialization of a prvalue. There is no guarantee that an xvalue refers to something expiring/temporary. We are better off defining things for what they actually are instead of how we expect them to be used.

# Acknowledgements

 - cppreference's [page](https://en.cppreference.com/w/cpp/language/value_category) on value categories was invaluable for this writeup.
 - The [official drafts](https://www.open-std.org/JTC1/SC22/WG21/docs/standards) for the C++ specification were also an invaluable resource.
 - Arno Schödl's [talk](https://www.youtube.com/watch?v=s9vBk5CxFyY) "The C++ rvalue lifetime disaster" didn't specifically influence anything specific on this writeup but the time I spent afterwards ruminating on the lifetimes of rvalues was invaluable.
 - Arthur O'Dwyer's blog posts ([this](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/) and [this](https://quuxplusone.github.io/blog/2020/03/04/rvalue-lifetime-disaster/)) regarding C++ lifetime encouraged me to avoid using the term "temporary" when discussing rvalues. I would not have introduced the concept of "rvalues are latent" without these writeups.
 - Anders Schau Knatten's [talk](https://www.youtube.com/watch?v=hkyZ8L343cU) "lvalues, rvalues, glvalues, prvalues, xvalues, help!", while a talk I dislike overall, introduced the shocking edge case that a `std::string` can appear on the left hand side of an assignment operation which helped me learn about default copy assignment overload behavior.
 - "The C Programming Language" 2nd Edition by Kernighan and Ritchie, which I used to provide an "official" description of lvalues in C.
 - "The Main Features of CPL" by D. W. Barron, et al was helpful for discussing the history of the concept of an lvalue.
 - Bjarne Stroustrup's [document](https://www.stroustrup.com/terminology.pdf) '"New" Value Terminology' is useful to see motivation for why value categories were made so much more complicated.
