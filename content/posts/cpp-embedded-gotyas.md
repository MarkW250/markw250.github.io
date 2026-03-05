+++
date = '2026-03-04T20:00:00+01:00'
draft = false
title = 'Five "Gotchas" When Using C++ for Embedded Development'
author = 'Mark Wilson'
tags = ['C++']
[params]
    graphcommentid = 'cpp_gotchyas'
+++

## Introduction

The `C++` programming language provides excellent tools for abstraction, code readability
(when used correctly), and quality-of-life improvements for developers.

There are some caveats, however. Some of the power that `C++` provides is built on
hidden abstractions that low-level-bit-wranglers should be aware of.

This article highlights five issues that might surprise embedded firmware developers
who are starting their journey with `C++`.

## 1. V-Tables {#v-tables}


A common pattern in `C++` is to define a purely virtual interface and then inherit
from it for a concrete implementation.

This is a useful abstraction: for example, a GPIO interface class can be injected
as a dependency into other classes, or other modules can obtain an instance of
GPIO from a global manager.

It also makes testing easier, since any module that depends on the GPIO interface
can mock or fake a concrete implementation for unit tests.

For example:

```C++
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


The cost of this abstraction is a vtable. For each _implementation_ of `IGpio`,
the compiler generates a table of function addresses (the virtual table or vtable),
and each _instance_ of the class contains a pointer to that table.

This means each instance typically has an implicit 4-byte overhead on a 32-bit MCU,
and each implementing class requires its own vtable stored in memory.

For the example above, the object is generally laid out in memory like this:
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


The cost is generally small, and some compilers can optimize away vtables in
certain cases. Still, it's good to be aware that they exist.

## 2. Template Bloat


Templates are a powerful tool. They can produce convenient helper functions that
reduce code duplication and avoid function overloading when a function must work
with multiple types.

Modern compilers are generally very good at optimizing template code. However,
template bloat can be a problem: if code size is constrained (as it usually is on
embedded devices), it's a good idea to track code size and watch for changesets
that significantly increase it.

Measure, measure, measure!

## 3. Construction of Static and Global Objects


This can be particularly troublesome if your project uses a custom linker or
custom startup code.

A constructor of a `C++` object runs when that object comes into scope. For example:

```C++

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


What happens if you declare an object outside of any function scope (making it a
static or global object)?

```C++
void MyFunction()
{
    MyClass myObject; // <-- "Hello!" gets printed when the function is entered.
}

MyClass myStaticObject; // When does the constructor run for this object?

```


The answer is that `C++` requires a runtime environment to run global object
constructors—just as `C` requires a runtime to zero `.bss` and initialize `.data`.
If you use custom startup code and a custom linker file, you must ensure global
constructors are handled. Usually the toolchain's default startup code provides
this automatically when it supports `C++`.

The main "gotcha" for embedded developers is that the `C++` standard does not specify
the order in which global objects across translation units are constructed. If the
construction of one object depends on another object or on initialized hardware,
avoid static initialization. In general, prefer explicit initialization rather than
relying on static global constructors.

## 4. Debugging Made Hard (Sometimes)


Modern compilers are generally excellent at optimizing the abstractions that `C++`
provides. This is both a blessing and a curse.

If you rely on optimizations because of resource constraints, you may be unable to
build a full debug (no-optimization) version of your project. Debugging optimized
`C++` code can be difficult: templates may be inlined, and execution may jump between
arbitrary constructors or destructors (or indeed just completely skip them), 
which makes stepping through code challenging. Granted, the same can
occur in `C`, and an experienced developer can navigate the optimizations by
inspecting the assembly code together with the source code. Or, one can fall back to 
good old "printf" debugging.

## 5. Constructors and Destructors


At first glance, `C++` constructors and destructors are straightforward: a constructor
is called when an object is created (for example, with `new` or when it is declared),
and a destructor is called when the object is deleted or goes out of scope.

However, things can get complicated quickly.


First, with inheritance, constructing a derived class implicitly calls the base
class constructor. While this is sensible, long chains of constructors can be
hard to follow, so prefer shallow inheritance hierarchies and use inheritance
primarily for interfaces (as shown above in [point 1]({{< relref "#v-tables" >}})).

Second, in addition to "normal" constructors,the `C++` language defines several kinds of 
constructors and assignment operations: copy, move, copy-assignment, and move-assignment.
These are especially important when a class manages resources. The `C++` core guidelines offer 
[good rules to follow](https://en.cppreference.com/w/cpp/language/rule_of_three.html).


Why is this a "gotcha" for embedded developers? We often think about how the compiler
generates assembly and manages objects in memory in order to reason about the memory
usage and time a function will spend doing a function.For example, in `C`, it is clear
that passing a struct or other type to a function makes a copy; and to avoid the copy you
pass a pointer. In `C++`, object abstractions, constructors, and copy/move semantics
can make reasoning about what the compiler will produce a bit more tricky.

## Conclusion


This article highlights features and characteristics of `C++` that are especially
important to consider in embedded projects. With discipline and good practices,
these issues can be managed, and `C++` can be a powerful enabler for embedded
development.

