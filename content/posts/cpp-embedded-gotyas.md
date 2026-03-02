+++
date = '2026-01-29T19:02:02+01:00'
draft = true
title = "Ten Gotya's When Using c++ for Embedded Development"
author = 'Mark Wilson'
tags = ['c++']
+++

## Introduction

The _c++_ programming language provides some excellent tools for abstraction,
code readability (when done right), and quality of life enhancements for developers.

There are some caveats, however. Some of the power that _c++_ enables is provided on
top of abstraction that might be important for low level big wranglers to be aware about.

This article touches on 10 issues that might surprise embedded firmware developers who
are used to the "directness" of _c_.

## 1. V-Tables

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
};
```

The cost of this abstraction is a "v-table". For each _implementation_ of `IGpio`, 
the compiler will create a list of addresses of functions (the "vector table" or 
"vtable"), and each _instance_ of the class that implements will have an address 
to this list of addresses.

This means, each instance will have a "hidden" 4 byte cost (for a 32-bit MCU), and
for each class that implements it, there will be a _4 * i_ bytes cost (where _i_ 
is the number of implementations).

The cost is generally small, and do know that some compilers are pretty darn good 
at optimizing away "vtables" in certain cases. But it is still good to be aware
that it exists.


## 2. Template Bloat

Templates are a powerful tool. Among other uses, they can be used to make very convenient
helper functions that can reduce code duplication, or the need for function overloading
when you need a function to act on multiple types.

However, template bloat _can be_ real if you get too liberal with your template usage.
Again, compilers can be pretty good at optimizing templates as well, but remember
to always measure!

## 3. Static Construction

This can be a particularly nasty one, particularly if you find yourself in a project
that, for some reason, needs a custom linker and startup code. 

....

## 4. Debugging Made Hard

## 5. Exceptions

## 6. Constructors

## 7. The Hidden Self

```c
int main(); // This is some code
```
