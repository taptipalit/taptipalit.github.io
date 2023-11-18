---
title: "SVF’s Field-Sensitivity: Handling of GEP Edges"
layout: post
date: "2020-09-21"
---

Let’s now look at how SVF handles the simpler cases of field-sensitive analysis (without any cycle elimination and positive-weighted-cycles)  
  
Imagine a struct A {int idx; struct A\* next};  
And consider the C statement:  
p->idx = 10; // p is a pointer pointing to an object of type A  
  
Clearly, this statement involves computing the address of the field ‘idx’ of the type struct A. In LLVM IR, this is done by the Instruction GetElementPtrInst. It looks like -- 
  
%idx = getelementptr inbounds %struct.A, %struct.A\* %1, i32 0, i32 0

  
idx is a virtual register that contains the address of the field ‘idx’.  
  
The GEP instruction has its own dedicated page on LLVM website :) [https://llvm.org/docs/GetElementPtr.html  
  
](https://llvm.org/docs/GetElementPtr.html)Anyhow, what we need to remember is that GEP computes a pointer into a struct type. It doesn’t access any memory, it’s purely pointer computation. And the last index (0 in this case) denotes the index of the field being accessed (0 means the first field -- that is, ‘idx’)

Now, every time SVF encounters a GEP instruction, it creates a GEPEdge from the base pointer to the field pointer. The base pointer is the IR pointer to the object, and field pointer is the IR pointer to the field within the object. This means that there can be multiple GEPEdges for the same field, if the same field is accessed multiple times. It simply means that the address of that field was computed multiple times (and loaded into a virtual register).  
  
So, let’s take a look at what that looks like in the SVF Constraint Graph. Consider the C source code -- 

struct A {int idx; int \*p};

int main( void){

    int k = 10;

    struct A aobj;

    aobj.p = &k;

    return 0;

}

  
The IR instructions -- 

  %k = alloca i32, align 4

  %aobj = alloca %struct.A, align 8

  store i32 0, i32\* %retval, align 4

  store i32 10, i32\* %k, align 4

  **%p = getelementptr inbounds %struct.A, %struct.A\* %aobj, i32 0, i32 1**

  store i32\* %k, i32\*\* %p, align 8

The initial constraint graph is shown in Figure 5. The purple edge is the GepEdge (There are actually two types of Gep Edges, but for now, let’s talk only about the “Normal” GepEdge -- the NormalGepCGEdge). It does not show in the figure, but every Normal Gep Edge has the field index associated with it. And there can be multiple edges with the same index, if the field at the same index is accessed multiple times.

**Figure 5.** 

![](https://tpalitblog.files.wordpress.com/2023/11/field-cycle-1.png?w=1024)

Conceptually, a GepEdge is similar to a CopyEdge, with an added constraint on which fields are copied (similar to how Pearce et.al extended Andersen’s plain copy constraint). Figure 5 shows an example. Let’s focus only on nodes 12, 11, and 17. First the AddressEdge will make sure that pts-to-set(11) = {12}. Then, the purple GepEdge will act like a Copy Constraint, but instead of copying the whole pts-to-set(11), it’ll copy only the objects corresponding to the field of the GepEdge. In this case, the GepEdge corresponded to field index 1 (the second field), so it’ll copy only the second field of the ObjPN with id 12. 
  
Now, as you can see, there’s no such node in the graph (yet). So far, we just have a single node for the ObjPN for aobj. So SVF will create this ObjPN -- a GepObjPN, with base node-id as 12, and field-index as 1. Assume this GepObjPN gets a NodeId of 20, then points-to-set(17) = {20}.

If the second field is accessed again, a different GepEdge will be created, but solving it will result in the same GepObjPN node with NodeId = 20, being copied into the points-to-set of the destination of the GepEdge. Figure 6, shows the constraint graph at this stage.  

**Figure 6**  

![](https://tpalitblog.files.wordpress.com/2023/11/field-cycle-2.png?w=1024)

Note: The handling for the first field (0th index) is slightly different because the pointer to the struct and the pointer to the first field can be used interchangeably. I’ll talk about it later (if I get around to it).  

Note: The ObjPN corresponding to the whole object aobj is a FIObjPN (Field Insensitive Obj PN), but this is a bit of a misnomer. It does not mean that the object itself is field-insensitive. It just means that you can’t use this FIObjPN to talk about the individual fields of the object.  

Now, let’s take a look at how the StoreEdge from Node 9 to Node 17 will be processed. As discussed earlier, the StoreEdge will be reduced to a CopyEdge from the source of the StoreEdge, to all nodes that exist in the points-to set of the destination. Thus, it adds a CopyEdge from 9 to 20. This is the final Constraint Graph, shown in Figure 7. 
  
**Figure 7**

![](https://tpalitblog.files.wordpress.com/2023/11/conscg_final.png?w=1024)
