---
title: "Variant GEP"
layout: post
---

GEP Edges can be of two types -- Normal Gep Edges and Variant Gep Edges.

A normal gep edge is one where the index or offset is known. A variant GEP is a gep whose index is not a constant. For example -- 

struct Obj { int a; int b; int c; }

int\* ptr = &sobj;

for (int i = 0; i < 3; i++) {

int \*c = (ptr + i); // ← non-constant offset

}

In this case, a VarGepPE PAGEdge will be inserted for the source ValPN for ptr, and this VarGepPE is converted into a VariantGepCGEdge in the ConstraintGraph and solved in the following way --

Src → Dst  
  
Any object that the src can point to is first made field-insensitive. Then, these field-insensitive objects are added to the pts-to-set(Dst). 

## **VariantGeps and Arrays of struct**

The default MemoryModel is field-insensitive when it comes to arrays. All elements in an array are considered to be the same. Now if there is a variable based access to this array, it’ll result in a VarGepPE from this array. 

Consider this example --

struct Obj { int\* a; int\* b;};

struct Obj objects\[2\];

struct Obj\* optr = objects;

for (int i = 0; i < 2; i++) { **optr\[i\].b** = &bint; }

In this case, the variable i is being used to index into the array objects, via the pointer optr. The IR for the highlighted part will be as follows --

for.body:                                         ; preds = %for.cond

  %1 = load %struct.Obj\*, %struct.Obj\*\* %optr, align 8

  %2 = load i32, i32\* %i, align 4

  %idxprom = sext i32 %2 to i64

  **%arrayidx = getelementptr inbounds %struct.Obj, %struct.Obj\* %1, i64 %idxprom**

  **%b = getelementptr inbounds %struct.Obj, %struct.Obj\* %arrayidx, i32 0, i32 1**

  store i32\* %bint, i32\*\* %b, align 8

  br label %for.inc

There are two gep pointers involved in computing the address of optr\[i\].b. The first one computes the address of the i-th object in the array, and the second one computes the address of the field ‘b’, within the struct. 

The Constraint / PAG graph considering only the first gep is shown in Figure 1. 
  
We’d expect the GEP edge for the second gep instruction to be a normal gep edge because the offset is constant (1), but because the source of this gep is originated from a VarGep, the second GEP edge also becomes a VarGep. Logically it makes sense. The array itself is element/field-insensitive, so it’s impossible to distinguish between fields within the elements of this array.

This causes much imprecision in apr-hook framework for httpd. Figure 2 shows the Constraint Graph after the second gep edge is added.

**Figure 1**

**![](https://lh7-us.googleusercontent.com/WdtYIttfz1y0BA9g-OZMQQkUu_An-cbNi_1O9MqujUEdnv7I2XWVHeEqk0S2W8kOr6b167FmxVp3ntLn9XyUVKLsCWSeDPTpDa9is8rrYkQe-GcJr4NZwDo9_gXbfWgVQIpOFOfy_jJkH2rcuPIkJg)**

**Figure 2**

![](https://lh7-us.googleusercontent.com/jPHNZFELrDdRgmThPzgo5BdrnMtpdcDnwZkK4BbCK47zt0lS4c8-UzYydRcupPgV_4lIZLL2nMxmDOUstbG_YuPJIakEe8-Cb5gllAvbAEz2oeIqID8s5GRwg2jDILbkIq9WZfvmLNZmskBG3p9CQA)
