---
title: "Translation of C language to LLVM IR"

layout: post
date: "2020-09-21"
---

Anyhow, let’s consider a different language. Consider the LLVM IR for example. The LLVM IR is also a language that has pointers and dereferences to pointers. 

1. The Load instruction, loads a value into a virtual-register from a memory location. This is like loading via a pointer dereference, similar to the p = \*q constraint, a.k.a, the Dereference Constraint. 
2. The Store instruction, stores the contents of a virtual register to a memory location. This is similar to doing a store via a pointer dereference, similar to the \*p = q constraint, a.k.a the Assign Constraint.

  
Another thing to remember is that because the LLVM IR is closer to machine code, it makes very memory access explicit. Unlike C language, which hides one level of memory access -- what I mean is that, when we do c = 10; it accesses a memory location, but we don’t call that a pointer. But in LLVM IR world, a pointer is basically anything that points to memory. And the C statement ( c = 10;) will be translated to a StoreInst which involves a pointer to the memory location pointed to by c.

One handy rule to translate C statements into IR is to count the number of memory reads and writes. 

For e.g. a = \*ptr; This involves, first loading the value at the address of ptr. This value is a pointer, then we load the value at this pointer. Then we store this value into the memory location of variable a.  
  
  %0 = load i32\*, i32\*\* %p -- First load to load the ptr

  %1 = load i32, i32\* %0 -- Second load to load value @ ptr

  store i32 %1, i32\* %a -- Store value into var a

Key thing to remember here is that the IR \*knows\* the memory addresses of various variables, but not its value (what it holds). It knows the address of variable a, but not what it contains. It knows the address of ptr, but not what it contains (which is another pointer). So it needs the extra load compared to C source code. C language hides this from us. When we write the name of a variable, it implicitly means “the value” of the variable.
