---
title: "Cycles in the Constraint Graph"
date: "2020-09-21"
layout: post
---

One of the common optimizations in Andersen’s points-to analysis is cycle elimination. At a source code level, consider the following pointer assignments -- 
  
int a = 100;  
int \*p, \*q, \*r;  
p = &a;  
q = p;  
r = q;  
p = r;

  
This represents a cycle, from p → q → r → p. This is a very simple example, and these cycles can happen across functions, and complex situations, so it’s useful to be able to eliminate these cycles.
  
The key point about cycles is that all nodes in a cycle share the same points-to set. So, the analysis can be optimized by replacing all nodes in a cycle by a single node. 

Now let’s look at SVF’s implementation.

Note that only copy constraints can cause cycles. In SVF world, the copy constraints are represented by CopyEdges in the constraint graph. Remember, that during solving the LoadEdges and the StoreEdges, CopyEdges are introduced. These CopyEdges are introduced to make it easier to track cycles. So, anytime a cycle is introduced during solving, it shows up as a cycle of CopyEdges, and all nodes that are part of this cycle can be merged/collapsed into a single Node.  
  
Now, let’s take a look at a simple example of a cycle -- int \*p, \*q; p = q; q = p;

This is the same example that we were looking at earlier. Figure 3 shows how the cycle of Copy Edges is created in this case. The cycle is nodes 26 → 12 → 22 → 14 → 24 -> 16 ->26. So all of these nodes will be collapsed into a single node. This creates the final constraint graph that looks like Figure 4, where all of these nodes are collapsed into node 26. 

**Figure 3**

![](https://tpalitblog.files.wordpress.com/2023/11/cycle-1.png?w=1024)

**Figure 4**

![](https://tpalitblog.files.wordpress.com/2023/11/cycle-2.png?w=1024)

Note that each Copy Constraints in the C code, is translated into two constraints in the IR -- a Deref/Load constraint, and a Assign/Store constraint, and the effect of solving the two Copy Constraints in the C source code is the same as the effect of solving the two Load and two Store constraints in the IR.
