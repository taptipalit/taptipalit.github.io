---
title: "Equivalence of analysis in C Source Code and IR"

layout: post
date: "2020-09-21"
---

The LLVM IR is a representation of the C Source Code. But the constraints generated for the statement in C and in IR might be different. 

For example, p = \*q; is a single constraint of Deref/Load type in C, but results in two Load constraints and one Store constraint in IR. 

But this doesn’t mean that the result of performing the points-to analysis on C source code is different from performing it on the IR!!! In IR, you just have a more finer view into the memory operations, but eventually, the points-to sets for a memory location / variable will be the same whether you do it in C or LLVM IR.
