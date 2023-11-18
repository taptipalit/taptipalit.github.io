---
title: "Andersen Wave Difference Algorithm"

layout: post
date: "2020-09-22"
---

This is the variant of Andersen’s algorithm that seems to be the “default” in SVF. If you use -ander, this is the analysis that is run. I’ll describe this algorithm in short.

The key idea that differentiates this from vanilla-Andersen’s analysis is that instead of propagating the full points-to set, for each CopyEdge, at each iteration, it propagates only the difference between it’s points-to set at the end of the previous iteration, and, its points-to set at this iteration.

In order to do this, the constraint graph must not have any cycles, and must be sorted in topological order.

At a high level, each iteration is divided into three phases: 

**Phase 1:** First phase of each iteration is to collapse all cycles, and sort the graph in topological order. **Phase 2:** Then, process the Copy constraints/edges and propagate the difference in the points-to sets up the constraint-tree. 

**Phase 3:** Then, process the Load and Store constraints/edges and add the new Copy Edges.

If a new Copy Edge is added during Phase 3, then repeat.

The worst-case complexity of this approach is the same as vanilla-Andersen, but because it propagates only the difference in points-to sets, it’s faster in the average-case. And also, the Phase 2 and Phase 3 can be parallelized ( according to the paper [http://compilers.cs.ucla.edu/fernando/publications/papers/CGO09.pdf](http://compilers.cs.ucla.edu/fernando/publications/papers/CGO09.pdf))

Note: Because Phase 2 and 3 requires an acyclic graph, any positive-weight-cycles are collapsed and the struct object is made field-insensitive.
