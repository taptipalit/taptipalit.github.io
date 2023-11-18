---

title: "Pointer Analysis and Undefined Behavior in C programs"

layout: post
date: "2020-11-24"
tags: 
  - "pointer-analysis"
  - "undefined-behavior"
---

Recently, I came across the question --- can Pointer Analysis algorithms such as Andersen's algorithm and Steensgaard's algorithm, could correctly detect undefined behavior (such as buffer overflow) in C programs? If yes, how? And if no, why not? And if it can't how does it affect soundness of pointer analysis? This is a very interesting question and I'll try to share my thoughts in this post.

I believe the key to answering this question is to not classify behavior of C programs as defined or undefined when thinking about pointer analysis, but to think of pointer analysis as answering the question -- which objects can a pointer point to during execution of the program as intended to by the programmer?  
  
Also, pointer analysis operates on an abstract model, it operates with limited knowledge of the full program. Moreover, some "knowledge" isn't available at all at static analysis time at all (e.g. lengths of buffers read over the network, etc)  

In light of this, let's take a look again at two interesting undefined behavior in C programs that affects pointers.

1. Integer to Pointer casts  
      
    Casting an integer to a pointer, is classified as "implementation-defined" in the ANSI C standard (see [here](https://wiki.sei.cmu.edu/confluence/display/c/INT36-C.+Converting+a+pointer+to+integer+or+integer+to+pointer)). However, identifying the program points which perform this operation is trivial. Therefore, the pointer analysis implementation, on encountering an integer to pointer cast can easily fall back to saying, "this-pointer-can-point-to-ALL-objects-in-the-program", thus maintaining soundness, but losing all precision for that particular pointer. The SVF implementation does exactly this by modeling all pointers which are created from integers, to point to a "black hole object".  
    
2. Buffer overflows  
      
    Detecting buffer overflows however is a different challenge. Usually, buffers are traversed via a loop that looks something like this (even if it is using Libc functions such as memcpy etc, within memcpy there is a loop that is logically equivalent to this):  
      
    `for (int i = 0; i < SIZE; i++) {   ptr++ = buff++;   }`  
      
    Now, determining the bounds for this loop might be simple, but in general, SIZE might have been the result of a pointer dereference, or even been read from over the network! This type of bounds analysis on loop counters is very challenging, if not impossible to perform at analysis time. Also, note that even if it could be determined at analysis time whether the pointer would go out-of-bounds, it could almost **never** be determined which object would reside at the out-of-bounds location, as **this information is dependent on the heap allocation algorithm being used, order of allocation**, etc, etc.  
      
    So most pointer analysis implementations that I have seen, assume that a pointer which points to within an object, at the start of a loop, remains within the body of the pointer during all iterations of that loop.  
      
    From a security perspective, this is actually useful when detecting malicious buffer overflows. We can determine at analysis time which object a particular pointer can point to and for each iteration of the loop, we can instrument the program to insert a check that ensures that the pointer still points to the right object.  
      
    The only situation where I can see pointer analysis \*not\* being sound, is if there is intentional buffer overflows -- that is the same pointer is incremented (or decremented) in such a way that it accesses adjacent objects that have no relation to each other (not part of a C array, for example).  
      
    

I'm sure there are other undefined behavior of pointers that cause all sorts of intriguing questions about how they can be correctly modeled and analyzed! Super exciting stuff! :D
