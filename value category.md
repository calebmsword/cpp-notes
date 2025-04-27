Any expression in C++ can fall into one of a small number of categories called **value categories** of which the two most important are **rvalues** and **lvalues**. All expressions are either an rvalue or lvalue. This terminology is often criticized, and rightly so. First of all, they should be called "expression categories", as they categorize expressions, not values, and we should have "l-expression" and "r-expression" categories instead of rvalue and lvalue. Furthermore, lvalue and rvalue brings to mind "left value" and "right values", and you might think there may be some significance there. The sensible analogy to "left" and "right" is that lvalues and rvalues are opposites; an expression cannot be both an rvalue and an lvalue at the same time. Trying to make any further connection between leftness/rightness and lvalue/rvalue only leads to confusion.

To discuss the specifics of value categories with we will first present their historical evolution.

### Before C++11

Two defining characteristics of expressions are **value**, which is the actual value the expression computes, and an **address** which is the physical location in memory where the value is stored. (Expressions in C++ also have **types** but that is not signficant for this discussion.)  Despite what the name suggests, a "value category" tells us something about the __address__ of any given expression:

 - lvalues were expressions whose addresses are guaranteed to made available to the programmer. Hence, we will say that lvalues are **locatable**.
 - There is no guarantee the address of an rvalues will be made available to the programmer; Hence, we will say that rvalues are **latent**.

Some examples will elucidate:

 - int a = 3;
 - a; // this is an lvalue
 - the data stored in a is accessible to us with a named variable. in general, names of things represent locatable data in C++. This includes names of objects, functions, etc.
 - since the data is stored in a variable, the line of code after the expression "a" still has access to the location in memory which stores a's data. This is why & is defined for a, as it is safe to reveal this address to the programmer. That location in memory will have the data in a as long as a remains in scope.

 - 7; // this is an rvalue
 - a contrived example, but bear with me. When we write 7, the integer is whipped up and represented somewhere, but since it is not stored in a variable, it will not be made accessible to the programmer in future lines of code. While we can get the same *data* by writing `7` again in the next line of code, this data will be represented in a new address in memory. The compiler is allowed to trash the value at some indeterminate machine instruction sometime after this line of code, hence it is unsafe to reveal this address to the programmer. the address is made latent.

 - int a = 3; // '3' here is an rvalue.
 - integer literals are, in general, rvalues. While rvalues are latent, their address can be saved IF they stored into a variable. In this case, the location in memory for `3` will stored in the variable a, and the lifetime of the data for the literal is extended such that it shares the lifetime of the variable a. This is why we use the term "latent", as it is strictly possible for the address in memory for the rvalue to be saved elsewhere through assignment. This address is made available indirectly through the variable.

Note: It is a common misconception that lvalues and rvalues indicate the lifetime of data; one might think that lvalues are expressions whose data has persistent lifetime, and rvalues are expressions whose data has temporary lifetime. Obviously from the third example above, it is wrong to assume that rvalues represent data that will "vanish" by the next line of code. The term "temporary" needs to stop being used by the C++ community when they discuss rvalues as it often leads to this misconception. In general, do not associate value category with lifetime.

In C++11, the standards committee generalized the concept of "locatibility" into what they called "identity" and wanted to categorize expressions whose evaluation corresponds to an object's identity. An object has "identity" if the evaluation of the expressions offers some mechanism to the programmer to determine an object's uniqueness. Obviously, all "locatable" expressions (lvalues) have identity since the & operator can be used with them to find an object's identity, but an expression could also carry identity if its __data__ was guaranteed to identify an object. This means it is strictly possible for an expression to be latent (an rvalue) and still have identity. Such an expression manifested in the language with the rvalue reference cast introduced in C++11. Casting an object into an rvalue reference results in latent data (if I don't store it in a variable the result of the cast won't be accessible in subsequent lines of code) but since it is a reference it is an alias to an object and guarantees us the identity of that object. The C++ standards committee felt this expression was best represented by a third value category, and the pre-existing value category taxonomy was salvaged by changing the meaning of the word "rvalue":
 - nearly all of what we used to call rvalues are now called "prvalues" ("pure rvalues"),
 - what we used to call lvalues are still called lvalues,
 - an rvalue reference cast expression of an object type is of a new value category called "xvalue"
   - a function which returns an revalue reference to an object type is also considered an xvalue.
 - rvalue is now an umbrella term: xvalues and prvalues are specific types of rvalues.

There also exists the less useful umbrella term glvalue, short for "general lvalue", which refers to either xvalues or lvalues (yes, this means an xvalue is both a glvalue and an rvalue). Hence glvalues are all expressions which carry identity. As a result of this change, everything that was an lvalue before was still an lvalue and everything that used to be an rvalue was still an rvalue, but now there two specific and mutually-exclusive types of rvalues.

With this new change, a new constructor overload called a "move constructor" could be created for an object which takes an rvalue reference to object whose type is that of the class for that constructor. A move constructor is called if an rvalue or xvalue of the correct object type is given to the constructor, and unlike const lvalue references, rvalue references are mutable, meaning the move constructor can modify the object given to it in order to perform its task. (It is expected that the user implements a move constructor such that the new constructed object "moves" the resource(s) from the given object so that ownership of the memory/mutex/file/etc associated with the given object is exchanged with that of the newly constructed object. While this is a mere convention, it would be considered a mortal sin if your move constructor did anything other than this.) As such, rvalues are now considered "movable". lvalues have identity and are not movable, prvalues have no identity and are movable, and xvalues have identity and are movable.

The new term xvalue was originally introduced without any meaning. I prefer to think of xvalue as being short for "cross value" since an xvalue contains a cross of some characteristics from lvalues and some characteristics from pre-C++11 rvalues. There is a small class of expressions that were rvalues before C++11 that are now also xvalues, but the rvalue cast expression and the function which returns an rvalue reference are by far the most important and common examples encountered in practice so it will what we will emphasize in the current discussion.

C++17 added more wrinkles to the taxonomy. Before C++17, we could have a prvalue which represented a latent object (for example, a function call of a function whose return statement calls a class constructor). However, if a prvalue is of a class type, it no longer represents an object but instead acts as a "free coupon" for that result object. Hence there is no expensive object copy when a prvalue result is returned by a function (since C++ is a pass-by-value language), we simply "xerox the coupon" for the actual resultant object. The specification demands that, at some point, the object is materialized (the coupon is exchanged) into the actual result object, and which point the specification also demands that the prvalue is converted into an xvalue.

With this new feature, prvalues of nonprimitive types are no longer moved from. When we pass a prvalue to a class constructor, the prvalue is eventually materialized into an xvalue, and that xvalue is moved from. Hence in C++17 onwards, it is no longer true that prvalues are movable.

In short:
 - if an expression can be given to the & operator it is an lvalue, otherwise it is an rvalue
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

I will mention that the standard says that xvalues are "eXpiring values", terminology I strongly dislike. Since most practical usage of xvalues is to call a move constructor or move assignement, the value represented by the xvalue is meant to be "moved from" and eventually discarded, which I suppose what this terminology is supposed to mean. This hinges on the programmer adhering to convention, however. The object pointed to by an xvalue won't necessarily be discarded after the xvalue is moved from, so it is bad to use terminology which suggests the lifetime of the data represented by the xvalue. This also assumes that a move constructor that particular type actually exists and is implemented in the expected way. We are better off defining things for what they actually are instead of what we hope they should be.

What follows is a precise categorization of all expressions told in a format extremely similar to cppreference's page about value categories.

lvalue
 - names of variables (or functions, templates, data members)
 - all assignment expressions (=, +=, *=, etc)
 - an expression using the "dereference" operator (*)
 - string literals
   - the specification demands that object referenced by a string persists through the lifetime of the program (the terminology they use is "static storage"), so it is safe to make strings locatable. Many implementations will create only one object if you declare two string literals with the exact same content.
 Key properties:
  > & is defined
  > can be used as left operand of assignment operators, but only if value is modifiable

prvalue
 - any non-string literal (7, 3.3f, etc)
 - arithmetic expressions (+, -, *, %, etc)
 - logical expressions (&&, ||, etc)
 - comparison expressions (<, >, >=, etc)
 - enumerator
   - enumerators are evaluated at compile time. the resultant machine code is indistinguishable from C++ code which uses an integer literal instead of an enum.
 - lambda expression
 Key properties:
  > & throws
  > cannot be used in left operand of any assignment
  > can be used to initialize const lvalue reference (const MyClass& myRef = <prvalue>;)
  > can be used to initialize rvalue reference (MyClass&& myRef = <prvalue>;)
  > function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function

xvalue
 - a cast expression to rvalue reference to object type (static_cast<MyClass&&>(myClassInstance))
 - or a function that returns an rvalue reference to object type (std::move(myClassInstance))
 Key properties:
  > & throws
  > cannot be used in left operand of any assignment
  > can be used to initialize const non-rvalue reference (const MyClass& myRef = <xvalue>;)
  > can be used to initialize rvalue reference (MyClass&& myRef = <xvalue>;)
  > function overload defined for rvalue reference parameter is used, if defined, if passed as argument to that function


=================================
 Additional cases: 
=================================
functions:
 - if the return type is an lvalue reference, then the function call is an lvalue
  - (I find this decision extremely troubling since this creates a situation where the return value of a function is treated as a value with identity, even if the the lvalue reference points to something in the function that goes out of scope once the function completes execution. As such lvalue references returned by functions almost always induce undefined behavior. This greatly vexes me.)
 - if the return type is an rvalue reference, then the function call is an xvalue
 - otherwise, prvalue

comma operator
 - the value category of the final comma-separated expression is the value category of the comma expression

ternary
 - the value category of the resulting branch is the value category of the ternary expression

address-of operator (&)
 - a prvalue.
 - one might argue that this represents identity since the value of an address-of expression is the literal address of some object. However, it is possible to get the address of something that is later deallocated and as such we cannot guarantee forever that the result of a & operator is always the identity of an object.

subscript (a[n])
 - an lvalue if one operand is an lvalue, an xvalue if one operand is an rvalue
  - Since a subscript is fully equivalent to a dereference (*(a + n)) an array lookup is getting the identity of something. This expression does not have guaranteed addressibility if a does not have guaranteed addressibility (which occurs if a is an rvalue), so it must be some type of rvalue. Hence before C++11 we see this must have an xvalue here. Post C++17, the language is specified to materialize whatever object is created in this situation so it cannot be a prvalue.
  - Q: hold on, why isn't a an xvalue if a is an rvalue?
  - A: the language does not distinguish between an array and a pointer of the type of an array. See my argument for why &a does not have identity. The same argument can be used to show that a pointer cannot guarantee identity.

pre-increment, pre-decrement (++a, --a)
 - an lvalue (it is addressable since the result of the expression is the new value of the variable)

post-increment, post-decrement (a++, a--)
 - a prvalue (it is temporary since the result of the expression is different from the value of the variable in subsequent lines of code)

a.m
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
   - classes are a generalization of C structs, and one non-enumerator data member can be accessed from another using pointer arithmetic (yes, this means there are situations where you can access private fields of an objects. very unfortunate language design.) hence data member access should be treated in the same manner as array access. See my disucssion about the subscript operator for more information.


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

p->m
 -non-function, non-enumerator member
  - an lvalue

 -non-function, enumerator member
  - a prvalue

a.*p, a->*p
 - an lvalue if a is an lvalue, an xvalue if a is an rvalue

function member access of object (a.f, p->f, a.*pf, p->*pf)
 - a prvalue. these are particularly restricted, they can only be used as the left-hand argument of a function call and the compiler will throw if you try to use this expression for anything other than a function call. (you could argue this implies the existence of an additional value category if you wanted to make C++ developers lives even harder).

this
 - prvalue
 - `this` behaves like a pointer to an instance of a class, but it cannot be reassigned and & is not defined for it. It seems to me that xvalue would be a more appropriate categorization then, oh well.

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
 - xvalue if casted to rvalue reference type (by definition)
 - lvalue if casted to lvalue reference type
   - (I really don't know why lvalue reference casts are not xvalues, for all the same reasoning that rvalue reference casts of objects are xvalues. However, the C++98 standard decided these were lvalues, and no changes to the taxonomy from C++11 was going to change something that once was an lvalue into an rvalue, so this is easy to explain from C++'s insistance on backwards-compatibility. The real answer is probably because move semantics would become even more confusing than they already are if both lvalue reference casts and rvalue reference casts could be moved from.)
 - prvalue otherwise

void expressions
 - void functions, casting something as void, and throw expressions are all prvalues, but these expressions are not allowed to be used as function arguments or used to initialize references. throw expressions may be used in either branch of the ternary operator. (you could argue this implies the existence of an additional value category if you wanted to make C++ developers lives even harder).


=================================
 Properties of value categories: 
=================================
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
