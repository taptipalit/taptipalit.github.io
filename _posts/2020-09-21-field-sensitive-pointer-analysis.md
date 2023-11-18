---
title: "Field Sensitive Pointer Analysis"

layout: post
date: "2020-09-21"
---

Field-sensitivity is the ability to distinguish between individual fields of a structure. Field-sensitivity introduces a problem with cycle-elimination, called Positive-Weighted-Cycle which I’ll talk about later. 

Field Sensitive Analysis was introduced by Pearce et.al. in Efficient Field Sensitive Pointer Analysis. 

Cycle detection and collapsing is a very important optimization that helps Andersen’s analysis. To recap, a cycle is of the form p⊇ q, ⊇ q r, r ⊇ p. In SVF’s world, it is a CopyEdge from q → p, r → q, and finally from p → r. And all the pointers in this cycle share the same solution, and therefore, can be collapsed.  
  
In order to make Andersen’s analysis field-sensitive, Pearce et.al extended the original constraint model with the concept of “weighted constraints” or “weighted constraint edges”. Every field in a complex struct type has an index with respect to the base of the object. And this index is assigned as the weight of the constraint. 

The constraint itself is of the form p ⊇ q + k, where k is the weight / field-index. 

In order to derive the solution of this constraint, let’s first recap the solution without the field-sensitivity. The derivation rule looks like --

p ⊇ q, q ⊇ {r} then, p ⊇ {r}  
  
Now, with the addition of the field constraint, we need to match the field index, in addition with the base-object (r, in this case). So we get --

p ⊇ q+k, p {r}, idx(s) = idx(r) + k, idx(s) <= end(r), then, p ⊇ {s}

Which is just a mathematical way of saying s is the object at an offset of k from r. Note that s is bounded by end(r), which is the maximum number of fields in the object.
