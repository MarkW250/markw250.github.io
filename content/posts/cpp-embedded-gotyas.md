+++
date = '2026-01-29T19:02:02+01:00'
draft = true
title = "Ten Gotya's When Using c++ for Embedded Development"
author = 'Mark Wilson'
tags = ['c++']
+++

## Introduction

The _c++_ programming language provides some excellent tools for abstraction and
code readability. It also enables easier use of some programming paradigms
that are otherwise difficult to achieve in _c_ (a typical
de facto weapon of choice for embedded programmers).

With great power, however, comes great responsibility. Here are 10 "gotya's" that
one should be aware of when using _c++_ for embedded development.

## 1. Virtual Functions

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

// In Led.cpp
class Led
{
    public:
        void init(IGpio& gpio)
        {
          m_gpio = gpio;
        }

        void turnOn()
        {
            m_gpio.setState(State::On);
        }
    private:
        IGpio& m_gpio;
};
```

The cost of this kind of abstraction is that any instance of a concrete implementation of
the interface now contains a "vtable" with function pointers. This is how c++ inheritance
works - the compiler essentially turns the class into a structure of pointers like you would
in c to achieve a similar goal.

Why is this a "gotya"? One might not always realize the cost of speed for the indirection
and the cost of memory usage for the vtable. It may not be of greater
significance in a lot of projects, but sometimes, when each nanosecond of time and each
byte of memory counts, one might consider a different approach.

The important points to keep in mind are:

- How many methods does the interface define? Roughly, for a 32-bit MCU, each interface will increase
  the vtable of each instance by 4 bytes.
- How many of these instances am I expected to have? If you're going to have 10 GPIO instances,
  each with 4 methods (init, setState, getState and toggleState); that's `4 * 4 * 10 = 160` bytes.
  Is this worth a simple GPIO abstraction?
- Am I calling a virtual method in a time critical path? Maybe one or even two indirections
  are fine, but if you use the pattern overzealously, you may end up with a slow application.

If these are problems for you, one can consider using an alternative such as
a classical facade to abstract the hardware:

```c
// Gpio.hpp
// Initializes all GPIO's for the project
extern void Gpio_Init();
// There are no "instances" of GPIO objects. Instead, they would
// be managed either by a classic switch case statement (for example).
extern void Gpio_SetState(GpioInstance instance, State newState);
```

## 2. Template Bloat

Templates are a powerful tool. Among other uses, they can be used to make very convenient
helper functions that can reduce code duplication, or the need for function overloading
when you need a function to act on multiple types.

However, template bloat _can be_ real if you get too liberal with your template usage.

## 3. Static Construction

## 4. Debugging Made Hard

## 5. Exceptions

## 6. Constructors

## 7. The Hidden Self

```c
int main(); // This is some code
```
