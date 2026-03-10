+++
date = '2026-03-09T20:00:00+01:00'
draft = true
title = 'From Compiler to CMake. A Build Toolchain Abstraction Story'
author = 'Mark Wilson'
tags = ['CMake', 'make']
[params]
    graphcommentid = 'make_to_cmake'
+++

## Chapter 0: Programmers are not Martial Artists

Once upon a time - a time not so long ago in relative human history, but long enough ago 
that it was way before my time as a firmware engineer, legend has it that programs were
written by [punching holes in paper](https://www.ibm.com/history/punched-card). 

<!-- TODO: Insert image of dude literally punching paper -->

The overlap of people who programmed computers and people who like to punch paper
though was not very large. And thus, we quickly moved on to programming computers
that can program computers.

<!-- TODO: Insert venn diagram of punchy ppl vs programmer ppl -->

I'm skipping a ton of historical details and steps here, of course, but please allow me 
to try get back to the main point of the story, and that is: how did we land up with a 
seemingly complicated tools such as CMake becoming necessary?

Once you know its origin and the problem it is meant to solve, I am hoping it will help
you use it more effectively, and understand its limitations.

I also want this to be an interactive story. I will provide as simple examples as
possible so that it is easy to follow along and feel the pain of our ancestors as 
they abstract their way into greatness.

## Chapter 1: The Creation of the Compiler

Once programmers got tired of punching paper, they moved on to making programs that can
make programs from a higher level language. First up was assembly, where you can at least
type out the instructions instead of punching them in paper. This was still laborious, 
error prone and often landed up with either extremely difficult to follow optimized
sets of instructions or easier to follow but very redundant sets of instructions.
So still not ideal, and thus higher level programming languages were invented to 
convert more readable syntax into instructions.

The programs that do this conversion are generally known as compilers. Again, I'm skipping
out on details. _Compiler_ is often an overloaded term, and there are many stages involved
in converting a high level language to the actual binary instructions that a processor 
can understand, and eventually build a binary that can be loaded onto a micro-controller
as a complete program or an executable file for Linux or Windows. The details will be
covered in future posts.

For this post, I want to call out two "main" steps in the complete process:
- Generating object files from source files (written in assembly, `C` or `C++`).
For the sake of this article, I will refer to this as _compilation_ and the 
tool (and its "sub tools") involved as the _compiler_.
- Linking all of the object files together to create a final application / binary file.
We will refer to this as _linking_ and the tool that does this as the _linker_.


<!-- TODO: Insert block diagram of compiler -->

## Chapter 2: The Invoking of the Compiler

So instead of punching cards, we get to write programs in a syntax that is somewhat
human readable, but also precise enough that the compiler can reliably produce instructions
in a (mostly) predictable manner.

Today, we can invoke the compiler in the command line. Let's try this now.

