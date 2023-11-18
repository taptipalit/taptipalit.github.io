---
title: "Points-to Analysis: Intro"
date: "2020-09-21"

layout: post
---

Points-to Analysis is widely used, but it’s tricky to understand. There’s a wide variety of notations used, leading to confusion. And if that wasn’t bad enough, the most popular points-to analysis tool SVF, operates on LLVM IR, while most descriptions of points-to analysis algorithms use examples from C code. This makes it difficult for readers to map the familiar C source code based examples to the functioning of SVF. 

This, and the following blog articles will describe things that I have learnt during my experiences with SVF and pointer analysis.

# **Andersen’s Algorithm**

It is an iterative, subset-based points-to analysis algorithm. I’ll describe the different ways to think about the constraints and derivation rules, that I’ve found to be helpful.

1. p = &q:  
    This is the Address-of Constraint.  
    Here the address of q is stored to p. So q ∈ p. Where p is the pts-to set for p.  
    
2. p = q:  
    This is the Copy Constraint.  
    Here both p and q are pointers (or pointers to pointers, or pointers to pointers to pointers … and so on). This is fairly intuitive. It means, the points to set of q is a subset of the points-to set of p.  
    In other words, p might\* point to whatever q might point to, in addition to whatever else, p might already be pointing to.  
    In other words, because q is a subset, we can denote p ⊇ q, denoting p is a superset of q. Some texts use this notation.  
    In other words, p ⊇ q, lx ∈ q, then,  lx ∈ p.  
    In other words, the points-to set of q, is copied into the points-to set of p.  
    Note\*: I use might because points-to analysis is imprecise and overapproximated.  
    Note: the alphabet is used to denote both the pointer / object as well as its points-to set. The meaning is clear depending on the context.  
    
3. \*p = q:  
    This is the Assign Constraint.  
    Here q is a pointer (or a pointer to a pointer or so on ..) and p is a pointer-to-a-pointer (or a pointer to a pointer to a pointer and so on …). Intuitively, it means that  the points-to set of q is a subset of each of the points-to sets of every element that is in points-to set for p.  
    In other words, every element that p might point to, might also point to whatever q might point to, in addition to whatever they might already be pointing to.  
    In other words, for every element e such that, p → e, e q.  
    In other words, \*p ⊇ q, lr p, lx ∈ q , then,  lx ∈ r.  
    In other words, the points-to set of q, is copied into every element in the points-to set of p.  
    
4. p = \*q:  
    This the Deref Constraint.  
    Here q is pointer-to-a-pointer (or a pointer-to-a-pointer-to-a-pointer and so on), and p is a pointer (or a pointer-to-a-pointer and so on). Intuitively it means that the points-to set of every element that q could point to, is a subset of the points-to set of p.  
    In other words, p might point to whatever, every element that is in the points-to set for every element that might be pointed to by q.  
    In other words, for every element e such that, q → e, p ⊇ e.  
    In other words, p ⊇ \*q, lr ∈ q, lx ∈ r , then,  lx ∈ p.  
    In other words, the points-to set of every element that q might point to, is copied into the points-to set of p.

The Andersen’s Algorithm, and practically all static analysis algorithms are language-independent. For example, all languages that supports pointers and allows for pointers to be assigned and dereferenced, will work with Andersen’s Analysis.

Also, note that C programs can contain more complex statements like \*\*p = \*\*\*q, or whatever. These have to be broken down into simpler statements via temporary variables in order to do the analysis.
