---
title: "SVF Implementation of Andersen’s Analysis"
layout: post
date: "2020-09-21"
---

This brings us to the SVF implementation of Andersen’s Analysis. SVF operates on the LLVM IR. And this presents some quirks. 

SVF represents the constraints as a constraint graph. As expected there are 4 types of constraints (Address-Of, Copy, Assign, and Deref). Only SVF calls them -- 

1. Address-of

3. Copy

5. Store

7. Load  
    

(In addition to these 4, SVF also has a fifth type of constraint to handle field sensitivity, that I’ll talk about later).

In the constraint graph, the memory objects and pointers are represented as nodes, and the constraints between them as edges. IR pointers are represented as ValPN nodes, memory objects are represented as ObjPN nodes. Every node, both ValPN and ObjPN has points-to set associated with it.  
  
SVF solves constraints by first reducing all Deref (Load) and Assign (Store) constraints to Copy Constraints. Check the underlined part in the Andersen’s Analysis for why this is reasonable. Reducing to Copy Constraints allows SVF to apply optimizations (such as cycle detection, that I’ll talk about later)  
  
Let’s take a look at how different LLVM IR instructions result in the creation of new nodes and edges in the constraint graph (Actually in the PAG graph, but let’s defer the discussion of PAG vs Constraint graph for later). 

1. AllocaInst: This instruction is used to create a local variable on the stack. When SVF sees this instruction, say p = alloca i32\*\*, corresponding to the C statement int \*p, it creates two nodes. One for the pointer p (a ValPN node), and one for the memory object (a ObjPN node). The memory object represents the memory associated with the variable.  
      
    An Addr Edge is added between the ObjPN and the ValPN, denoting the address-of relationship. When this AddrEdge constraint is solved, points-to set of the ValPN includes the ObjPN.  
      
    This is exactly the same thing as solving the Addr-of constraint. p = &q.  
      
    Though the language changes ( C → IR) the rules that guide the derivation of the constraints don’t change. A Copy Constraint derives in the same way in the IR, as in C.  
      
    For example, consider the bolded C statements from the simple C program, whose Constraint Graph is shown in the figure below -- 
      
    **int a = 100;  
    int \*p, \*q;**  
    p = &a;  
    q = p;  
    p = q;  
      
    Translated to IR -- 
    %a = alloca i32, align 4  
    %p = alloca i32\*, align 8  
    %q = alloca i32\*, align 8

  
This creates three ValPNs and their corresponding three ObjPNs. The Green edges are the Address Edges.  
  
Each Node in the ConstraintGraph has a unique id (19, 20, etc).  
  
**Figure 1:**

![](https://tpalitblog.files.wordpress.com/2023/11/conscg_initial.png?w=1024)

2. LoadInst: This instruction is used to explicitly load the contents of an IR pointer into a virtual register. When SVF sees this instruction, say %0 = load i32\*, i32\*\* q, it creates one ValPN node for the virtual register %0, and adds a LoadEdge between the ValPN node for q (not the ObjPN) and the ValPN node for %0. 
      
    Consider the bolded statement in the C program, 

  
int a = 100;  
int \*p, \*q;  
p = &a;  
**q = p;**  
p = q;  
  
This is translated to the IR instructions:  
  
**%0 = load i32\*, i32\*\* %p, align 8**

  **store i32\* %0, i32\*\* %q, align 8**  %1 = load i32\*, i32\*\* %q, align 8

  store i32\* %1, i32\*\* %p, align 8

Let’s look at the first Load instruction. In the above constraint graph it is represented by the Red edge from nodes 11 → 20. 
  
This LoadEdge represents a Deref constraint. As stated before, SVF reduces this to a Copy Constraint from all nodes that exist in the points-to set (source) to the destination of the load. So in the simplest case, LoadEdge will lead to the creation of a CopyEdge from the ObjPN that existed in the source ValPN’s pts-to set (added because of the AddrEdge), to the ValPN of the destination.  
  
\[Quick tip: Think about how you’d solve p = \*q. p is the destination, q is the source of this load. You’d first find all elements you know q to point to, (call them e1, e2_,_ etc) and then add copy constraints q = e1, q = e2, etc. The source is the e’s and the destination is q.\]  
  
Solving the two LoadEdges (11 → 20 and 13 → 22) results in adding the following CopyEdges (black edges). Note, that the points-to sets of the relevant pointers are denoted as {node-id} beside the pointers.  
**Figure 2**  

![](https://tpalitblog.files.wordpress.com/2023/11/andersen-1.png?w=1024)

3. StoreInst: This instruction is used to explicitly store the contents of a virtual register to a memory location. When SVF sees this instruction, say  store i32\* %1, i32\*\* %p, it creates an StoreEdge from the source ValPN (%1) to the destination ValPN (%p).  
      
    This StoreEdge represents an Assign Constraint. In the simplest case, this will be reduced to a CopyEdge from the ValPN of the source pointer, to the ObjPN in the pts-to set of the ValPN of the destination pointer (added because of the AddrEdge)  
      
    Let’s continue with the previous IR instructions -- 
      
    %0 = load i32\*, i32\*\* %p, align 8

 **store i32\* %0, i32\*\* %q, align 8**  
%1 = load i32\*, i32\*\* %q, align 8

 **store i32\* %1, i32\*\* %p, align 8**  
\[Quick Tip: In order to understand how SVF solves this, think about the assign constraint \*p = q. To solve this, you’d first find all elements e that exist in the points to set for p, and then add a Copy Constraint e = q.\]  
  
The three Store Edges add the three (bolded, dashed) black Copy Edges.  
  
**Figure 3**  
  

![](https://tpalitblog.files.wordpress.com/2023/11/andersen-2-1.png?w=1024)

  
  
CastInst: CastInst instructions do BitCasts and other casts and these result in CopyEdges in the LLVM IR. CopyEdges are solved in the same way as Copy Constraints in the Andersen’s analysis.
