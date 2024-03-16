---
title: Garbage Collection in Python
author: Chandra Kiran G
slug: garbage-collection-in-python
pubDatetime: 2024-01-07T04:06:31Z
draft: true
tags:
  - garbage-collection
description: Garbage collection is an essential step that ensures the efficient use of memory in a program. In this article, we'll look at the basics of garbage collection, its importance, and the various algorithms used to implement it.
---

# Garbage Collection

[[January 26, 2024 18:23:07]]

## Why do we need a Garbage collection?

You keep running your servers all the time. You want those six 9's availability. You want to optimise the cost. You want low latency. And what not? Though not completely, one of the contributing factors to achieving this is efficient memory management. Let's assume that you fetch 10,000 rows to respond to an API call (just "assume"). To access these rows in your program, you need to store them in memory. When we're done with our processing, we'll return the response to the user, completing our request-response cycle. But what happens to the memory allocated for those 10,000 rows? How can we be sure that this memory is freed before the next API call and that we can reuse it safely without any memory leak?

One way to do this is to manually release the memory used by the code. In programming languages ​​like C/C++, this is the only way to release unused memory. Remember those C functions `malloc()`, `calloc()` and `free()`? That's what I'm talking about. The memory allocated by `malloc()` or any other function must be freed using the `free()` function. It's not too hard to follow this, but it can get cumbersome very quickly as the code becomes complex. We need to call `free()` in the right places to avoid [dangling pointers](https://en.wikipedia.org/wiki/Dangling_pointer), [memory leaks](https://en.wikipedia.org/wiki/Memory_leak) or both (Refer to the figure below).

![[memory leak and dangling pointer.png]]

The other way, and the most widely adopted way is to use a garbage collector. Many modern programming languages use garbage collectors for automatic memory management. Let's now see what a garbage collector is and the various algorithms to implement it.

## What is Garbage Collection?

Garbage collection, according to [Wikipedia](<https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)>) is:

> A form of automatic memory management. The *garbage collector*
> attempts to reclaim memory that was allocated by the program, but is no longer referenced; such memory is called garbage.

This automatic process frees the user from having to manage memory manually. How does a garbage collector know what memory to free? The key is to figure out the _garbage_. A garbage is any chunk of memory that was allocated by the program and is not used anymore. The next thing is rather simple, cleaning the garbage. The first step is what makes the difference. There are various algorithms to achieve that (and many different variations too). Few of the most popular ones are:

1. Tracing Garbage Collection
2. Reference Counting
3. Generational Garbage Collection

Many garbage collector implementations use a combination of these.

## Tracing Garbage Collection

This is one of the earliest garbage collection algorithms, dating back to 1960. Also called Mark-sweep garbage collection, operates on the pointer reachability. It works in two phases:

1. Tracing phase or Marking phase: In this phase, a collector traverses the graph of objects in the heap. As it traverses the objects, it marks each object that it finds (live objects).
2. Sweeping phase: The collector examines every object in the heap and any unmarked object (dead objects) will be deemed as garbage and its space is reclaimed.

This is an _indirect collection_ algorithm. It does not detect the garbage but instead detects all the live objects and concludes that anything else must be garbage.

This garbage collection is performed in cycles and these cycles are often triggered when there is no sufficient space to satisfy the allocation request by the program.

### Naive Mark-sweep Algorithm

```python
def mark_roots(roots):
    worklist = []
    for node in roots:
        mark(node)
    sweep()

def mark(node):
    if not node.marked:
        node.marked = True
        for each child referenced by node:
            mark(child)

def sweep():
	for each node in heap:
	    if node.marked:
	        node.marked = False
	    else:
	        free(node)
```

This is a trivial depth-first search. It starts by iterating through _roots_, the objects that are directly reachable in our code. It then marks the current node and iterates through all the references that the node contains and marks each of the nodes that are reachable from the current node in a DFS fashion.

The sweep phase scans the whole heap and un-marks all the marked nodes, this is essential so that these nodes will be again scanned in the next mark cycle. Any unmarked node will have its space reclaimed by the collector.

Though simple and easy to implement, this algorithm has several disadvantages. One is _the stop-the-world_ nature of it. That is, the entire program's main thread must halt during both phases as mutations to the working memory might spoil the marking phase. The second one is the fact that it needs to scan the entire memory twice (one in thee mark phase, two in the sweep phase). The third one is the [fragmentation](<https://en.wikipedia.org/wiki/Fragmentation_(computing)>). Frequently clearing up the memory results in live objects being separated far from each other. This results in severely fragmented objects making it hard to move from one object to another due to various memory offsets. This makes traversal more expensive.

### Tricolor Abstraction

This is an efficient variation of the naive mark-sweep algorithm. This partially solves the _stop-the-world_ problem of the naive version. The algorithm works in the following way:

1. All nodes are initially placed in the white set, representing unmarked nodes.
2. As the algorithm progresses, the nodes that are currently being explored are moved to a grey set.
3. The nodes that have been already explored are moved to the black set.

So at any point in time, a node can be in exactly one of the three sets. This algorithm makes progress through the heap by moving in wavefront separating black objects from white objects. The algorithm is complete when the grey set becomes empty i.e. when don't have any more nodes to process. The nodes left in the white set are not referenced by either roots or intermediate objects, hence their memory can be reclaimed. This algorithm preserves an important invariant: At the end of each iteration of the marking phase, there are no references from black to white objects. So once the grey set becomes empty, we can immediately free the white set.

The main advantage of this algorithm is that it can perform "on-the-fly" marking by concurrently moving objects between the sets as and when the program's main thread makes changes to the objects. So the sweep phase can be called whenever required. This sweep phase may not be preceded by the whole mark phase as the objects are already partially or completely sorted into separate sets.

## Mark-compact Garage Collection

Before going into mark-compact garbage collector, let's briefly understand what fragmentation is, and the problems caused by it.

#### Fragmentation

Fragmentation in memory refers to the inefficient use of free space due to its division into non-contiguous blocks. Fragmentation occurs when the total free memory is divided into smaller blocks that are not adjacent to each other. This prevents the allocation of larger memory blocks.

Fragmentation is a big problem when we're allocating and deallocating memory dynamically. The number of non-contiguous blocks may increase when deallocating, making it difficult for the allocator to allocate a bigger chunk of memory.

To avoid fragmentation, allocators move the allocated objects to a new memory location where all the gaps are filled and used effectively. This phenomenon is called compaction. The advantages of this technique are:

- Quick allocation of new objects as the allocator does not have to search long for free space.
- Fast deallocation as an entire region of the memory can be freed quickly as most of the objects are contiguous.
- Since all the objects are very close to each other, the traversal can be very quick.

Mark-compact allocators compact the live objects in the heap to eliminate fragmentation ([external fragmentation](<https://en.wikipedia.org/wiki/Fragmentation_(computing)#External_fragmentation>), to be precise). Mark-compact allocators operate in two phases. First is the marking phase, which was already discussed above. The second is the compacting phase which involves compacting all live objects and updating their references to point to the updated locations. Most collectors using Mark-sweep also periodically employ Mark-compact.

Compaction can be done in one of the three ways:

- Arbitrary: Objects are relocated irrespective of their original order. This is fast and simple but leads to poor spatial locality.
- Linearising: Objects are relocated in such a way that the adjacent ones are placed next to each other.
- Sliding: Objects are slid to one end of the heap, squeezing out the garbage. This maintains the original allocation order in the heap. All modern mark-compact allocators implement this technique.

We'll discuss the following mark-compact allocator algorithms:

1. Two-Finger Compaction
2. The Lisp 2 Algorithm
3. Jonker's Threaded Compaction

All compaction algorithms are invoked as follows:

```python
def collect():
  mark_from_roots()
  compact()
```

### Two-Finger Compaction

This is a two-pointer (two fingers) and a two-pass approach that compacts objects arbitrarily. It is best suited for compacting regions containing fixed-size objects. The idea is simple: given the volume of live data in the heap, we know where the boundary (high-water mark) of the region will be after compaction. Live objects above this threshold are moved into the gaps below the threshold. Let's look at the algorithm now:

```python
def compact():
    end = relocate(heap_start, heap_end)
    update_references(heap_start, end)

# first pass: relocate live objects to the beginning of the heap
def relocate(start, end):
    while start < end:
        # find the first unmarked object or free space from the start
        while is_marked(start):
            un_mark(start)
            start = start + size(start)
        # find the last marked object from the end
        while not is_marked(end) && end > start:
            end = end - size(end)

        if end > start:
            current_live_object = end
            current_gap = start
            un_mark(current_live_object)
            move(current_gap, current_live_object)
            # forwarding address
            *end = current_gap

            # continue the process
            start = start + size(current_gap)
            end = end - size(current_live_object)

    return end

# second pass: update the references to the live objects
def update_references(start, end):
    # update the root references that point to the old location of the live objects
    for node in roots:
        ref = *node
        # if the reference points to the old location of the live object
        if ref >= end:
            # update the reference to point to the new location of the live object
            *node = *ref

    while start < end:
        for node in pointers(start):
            ref = *node
            if ref >= end:
                *node = *ref
        start = start + size(end)
```

The algorithm uses two pointers `start` and `end`. The `start` pointer starts from the start of the heap and advances till it finds a free slot. Whereas the `end` pointer starts from the end of the heap and retreats till it finds a live object. Then, the object pointed by the `end` is moved to the free space pointed by the `start`. The old address gets updated with the address of the new location, called forwarding address. The first pass is complete when the two pointers pass each other. At the end of the first phase, the `start` pointer points to the high-water mark region.

The second pass updates the pointers that referred the objects beyond the high-water mark with the updated locations of those referred objects.

Pros:

1. Simple and fast
2. No memory overhead

Cons:

1. Poor fragmentation with objects of various sizes
2. The order of objects in compacted area is arbitrary

### The Lisp 2 Algorithm
