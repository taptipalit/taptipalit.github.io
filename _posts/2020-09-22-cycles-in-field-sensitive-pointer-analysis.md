---
title: "Cycles in Field Sensitive Pointer Analysis"

layout: post
date: "2020-09-22"
---

In the original Field-Sensitive paper by Pearce, this problem was described as the Positive Weight Cycle problem. Let’s first look at this problem as formulated in the original paper.  
  
Consider the following C code, that has a void\* pointer.  

typedef struct { int \*f1; int \*f2; } aggr;

int main(void) {

    aggr a, \*p; void \*q;

    q = &a; **// q** ⊇ **{a}**

    p = q; **// p ⊇** **q**

    q = &(p->f2); **// q ⊇** **p + 1**

    // do stuff with q

    return 0;

}

This leads to a weighted cycle -- p → q, q -- 1 → p. Unlike non-weighted cycles, the nodes in a weighted cycle don’t share the same solution, and it can lead to infinite derivations. 

For example, 

q = {a}

p = {a}

q = {a, r}, where r is the field at the index 1 from the base object a.  
  
Then, because it’s a cycle, the derivation continues,

p = {a, r}  
q = {a, r, s}, where s is the field at the index 1 from the base object r.

So on and so forth. Pearce et.al. basically puts a limit to these derivations, and say that the maximum number of fields we’ll support is N.

In Andersen's WaveDiff algorithm, cycles cannot exist because there _must_ be a topological order of all nodes in the constraint graph. Thus, when using Andersen's WaveDiff algorithm, all nodes in such a _Postive-Weighted Cycle_ are collapsed and _all_ objects they point to are turned field-insensitive.
