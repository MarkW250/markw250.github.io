+++
date = '2026-03-04T20:00:00+01:00'
draft = false
title = "Ten Gotya's When Using c++ for Embedded Development"
author = 'Mark Wilson'
tags = ['c++']
+++

## Introduction

The _c++_ programming language provides some excellent tools for abstraction,
code readability (when done right), and quality of life enhancements for developers.

There are some caveats, however. Some of the power that _c++_ enables is provided on
top of hidden abstraction that are important for low-level-bit-wranglers to be aware about.

This article touches on 10 issues that might surprise embedded firmware developers who
may be starting their journey in _c++_.

## 1. V-Tables {#v-tables}

A common pattern in c++ is to define a purely virtual interface, and then inherit
from it for your concrete implementation.

This is a neat abstraction such that for example a GPIO interface class can either be injected
as a dependency to other classes, or other modules can "get" an instance of GPIO from
some global GPIO manager.

This also enables easy testing since any module depending on the GPIO interface can
mock or fake a concrete implementation to use for its unit tests.

For example:

```c++
// In IGpio.hpp:
class IGpio
{
    public:
        virtual void init() = 0;
        virtual void setState(State newState) = 0;
};

// In ConcreteGpio.hpp:
class ConcreteGpio : public IGpio
{
    public:
        void init() override;
        void setState(State newState) override;

    private:
        State m_currentState;
};
```

The cost of this abstraction is a "v-table". For each _implementation_ of `IGpio`, 
the compiler will create a list of addresses of functions (the "vector table" or 
"vtable"), and each _instance_ of the class that implements will have an address 
to this list of addresses.

This means, each instance will have a "hidden" 4 byte cost (for a 32-bit MCU), and
for each class that implements it, there will be a _4 * i_ bytes cost (where _i_ 
is the number of implementations).

For the example above, the object generally is laid out in memory like this:
```
Vector table for ConcreteGpio:
0x00001000: Address of ConcreteGpio::Init()
0x00001004: Address of ConcreteGpio::setState()
...
An instance of ConcreteGpio:
0x00002000: 0x00001000 (address of vector table)
0x00002004: m_currentState
...
```

The cost is generally small, and know that some compilers are pretty darn good 
at optimizing away "vtables" in certain cases. But it is still good to be aware
that it exists.

## 2. Template Bloat

Templates are a powerful tool. Among other uses, they can be used to make very convenient
helper functions that can reduce code duplication, or the need for function overloading
when you need a function to act on multiple types.

Modern compilers are also generally _really_ good at optimizing template code. However,
template bloat can indeed be a problem - if code size is a potential constraint (as it 
normally is on embedded devices), its always a code idea to try keep track of your code 
size and try to spot when a change set suspiciously increases code size significantly.

Measure, measure, measure!

## 3. Construction of Static and Global Objects

This can be a particularly nasty one, particularly if you find yourself in a project
that, for some reason, needs a custom linker and startup code. 

A constructor of a c++ object runs when that object comes into scope. For example:

```c++

#include <cstdio>

class MyClass
{
    MyClass()
    {
        printf("Hello!\n");
    }
};


void MyFunction()
{
    MyClass myObject; // <-- "Hello!" gets printed when the function is entered.
}

```

What happens, though, if you have an object declared outside of a function scope (making it a 
static or global object)?

```c++
void MyFunction()
{
    MyClass myObject; // <-- "Hello!" gets printed when the function is entered.
}

MyClass myStaticObject; // When does the constructor run for this object?

```

The answer is that _c++_ requires a runtime environment that will run global object constructors. 
Just like _c_ requires the runtime environment that will zero out `.bss` and initialize `.data` 
sections. And just like _c_, if you find yourself with a custom startup code and linker file, 
you need to take that into consideration. Often, you would use whatever linker and startup 
code is provided by your toolchain, and if it supports compiling _c++_ then it should provide
appropriate startup code auto-magically.


Now the main "gotya" for embedded folks here is that there is no specified order in which these 
objects will be constructed. 
Therefore, it is really important to keep in mind that if the construction of object relies on a 
different object already being constructed (or some hardware to already be initialized), then do not 
make these objects statically construct! In fact, I would argue that it is best to stay away from 
static objects in general. More on that in an upcoming article.

## 4. Debugging Made Hard (Sometimes)

I have mentioned more than once in this article that modern compilers are generally really good
at optimizing the logical abstractions that _c++_ provides. This is both a blessing and a curse.

If you rely on the optimizer to optimize your fancy pants _c++_ code because you have resource
constraints, then you might get to a point where you cannot build your project with
no optimizations (i.e. a full debug build). If you ever need to debug optimized _c++_ code, 
you are in for a fun time. From templates being inlined, to the code jumping to constructors and
destructors, it is sometimes difficult to step through the code sanely. Granted, it can
be similar case with _c_, and an experienced developer can navigate the optimizations by
inspecting the assembly code together with the source code. Or, good old "printf" debugging
could work if your environment provides some kind of output.

## 5. Constructors and Destructors

At first glance, _c++_ constructors and destructors are pretty straight forward: Constructor
is called when the object is created using the `new` operator or at the time it is declared, 
and the destructor is called when it is either deleted using the `delete` operator or when 
the object goes out of scope.

However, things can get complicated quickly. 

Firstly, when you deal with inheritance, and you construct a child class, the parent class's
constructor is implicitly called. Sounds reasonable, but believe me this can get out of hand
and often even a shallow chain of constructors can quickly eat up precious working mental 
memory when someone tries to decipher the code. This is one of the many reasons why I would 
recommend keeping inheritance extremely shallow when designing your code - preferably limiting 
it to defining interfaces (like shown above in [point 1]({{< relref "#v-tables" >}})).

Secondly, there are many types of constructors defined by the _c++_ spec: in addition to the
"normal" constructors, there are move constructors, copy constructors, copy assignment and
move assignment. These become particularly important to know about as soon as a class
is managing a resource. In this case, the _c++_ core guidelines offer 
[good rules to follow](https://en.cppreference.com/w/cpp/language/rule_of_three.html).

Why is this a "gotya" specifically for embedded developers? Well, often we are often thinking
about how a compiler will create the assembly and how it manages objects in memory so that
we have an idea of code size and code speed. For example, in _c_, it is very clear that
when you pass a `struct` to a function, a copy of the struct will be made. If you want 
to use less stack and save some processing cycles, pass a pointer to the `struct`. Its 
the same with all other types in _c_. In _c++_, the abstraction through objects and their
various constructors (and the way one can customize them), it is not always as clear cut. 

## Conclusion

This article points out some features and characteristics of _c++_ that are particularly useful to
be aware about in the context of an embedded project. Generally, however, these can be overcome,
and with proper discipline and training, _c++_ can be an incredible enabler in the context
of embedded projects.

