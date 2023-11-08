---
layout: post
title:  "Which Collection do I use in C#?"
date:   2023-11-08 18:09:00 +0100
categories: csharp
---

# Introduction

C# has a large amount of built-in collection types. With the move to .NET core, the compiler team also introduced a whole set of immutable, persistent collections.
These immutable collections are used heavily in the fully immutable and open source Roslyn compiler for C#, so they aren't just some academic's toy.

Without a full and comprehensive education in algorithms and data structes, it's really hard to know which collection to chose for which use-case.
Some frameworks try to hide this decision behind object-oriented facades. This way, the developer only needs to write code for a single item at a time.
Other applications move most if not all collection logic into the database layer. But sometimes, you still need to deal with collections of elements yourself.

The default in C# is a `List<T>` - a simple array-backed collection, known in Java as `ArrayList<T>` and in C++ as `std::vector<T>`.
And it's a good collection indeed. For a few elements (hundreds at most), it performs well for pretty much all use-cases.
But for more elements, some operations are slow: Specifically insertion and removal of elements not at the end, as well as checking if an element is contained in the list.

But performance is not the only concern: a `List<T>` is mutable, meaning that anyone with a reference to it can add and remove elements at will.
This property can make it hard to reason about code. It prevents you from making assumptions that can save defensive copies.

It's my firm belief that any programmer who writes any kind of system needs to know their collections. Which is why I'm writing this post as reference material.
The contents of this post are not limited to C#; it discusses collections implemented in many programming languages. C# is just my language of choice to discuss the specifics.

# Landau Notation

This post talks a lot about performance. 
As a consequence, it's really helpful to understand the basics of *Landau Notation*.
This notation describes the worst case performance characteristics of code based on the size of some inputs (= the size of the collection), while disregarding implementation details.
The exact mathematical definition isn't important when talking about collections, so instead I'll just describe the cases used in here:

- `O(1)` means that the code runs in constant time
  - It might be slow, but it runs in the same time no matter how large the collection gets.
  - E.g. assigning an element to an index in an array or a list: `list[i] = item`
- "amortized" `O(1)` means that the code runs in constant time *on average* 
  - The operation might take a while sometimes, but if you repeat it often, it's independent of the collection size. 
  - Think of it like a garbage collector: it usually does nothing interesting, but sometimes it blocks to cleanup some memory.
  - E.g. adding an element to the end of a list: `list.Add(item)` (why later)
- `O(n)` means that the runtime of the code increases linearly with the size of the collection.
  - The larger the collection, the slower the operation.
  - E.g. checking if a list contains an element: `list.Contains(item)`
    - in the worst case, the code needs to check every element in the list until it potentially finds the item.
    - A list with one element might take some constant time for loop setup, checking, returning etc.
    - But a list with `n` elements will take this constant time + `n` times the time it takes to check for equality.
- `O(log n)` means that the runtime increases logarithmically with the size of the collection
  - Whenever the size of the collection doubles, a constant factor is added to the runtime.
  - In practice, this is fairly close to `O(1)`: ld(1,000) ≈ 10, and ld(1,000,000) ≈ 20
- `O(n log n)` is short notation for `O(n × log(n))`
  - This is the proven worst case optimum for sorting a collection of elements
  - Close to `O(n)` in practice, but supports fewer number of elements until it gets "slow"
- `O(n²)` means that the runtime increases quadratically with the size of the collection
  - Think of this as two nested foreach loops over the same collection.
  - E.g. "naive" sorting ("insertion sort"): for each element, go through the list and insert it at the right spot, moving all successive elements back
  - This should really be avoided. Consider that 1,000² = 1,000,000, so this can get slow even for smaller collections

# Relevant Operations

Different collections support different operations. Some support special operations, and some don't support even trivial operations.
This section names some operations and explicitly defines them, so that we can compare different collections for these.

- `append(x)` - adds an item to the end of the collection
- `prepend(x)` - adds an item as the first element of the collection
- `add(x)` - adds an item anywhere, e.g. in unordered collections
- `removeFirst()` - removes the first element
- `removeLast()` - removes the last element
- `remove(x)` - removes the item from the collection when present
- `contains(x)` - checks whether the item is present in the collection


# Of `IEnumerable<T>` and when to use it

TODO

# Immutability & "Persistent" Collections

A (reference to an) object is called *immutable* if you cannot modify that object's state.

In C#, the `IReadOnlyList<T>` and `IReadOnlyCollection<T>` interfaces offer a reduced API that doesn't allow the holder of the reference to mutate the collection in any way.
An `IReadOnlyCollection<T>` is an `IEnumerable<T>` but with a `.Count` property that runs in `O(1)`.
An `IReadOnlyList<T>` is an `IReadOnlyCollection<T>` which also allows accessing elements by index in `O(1)` or `O(log n)`.
All allocated, strict (non-lazy) collections in C# implement at least `IReadOnlyCollection<T>`.

But be careful: if you have a variable of type `IReadOnlyCollection<T>`, it might point to a collection that's still mutably referenced from somewhere else.
This means that you cannot simply assume that a given `IReadOnlyCollection<T>`'s contents never change, unless you control its origin.

> 💡 Prefer using `IReadOnlyCollection<T>` or `IReadOnlyList<T>` over concrete collection types if you don't need to modify the collections!

For fully and definitely immutable collections, C# offers a selection in the `System.Collections.Immutable` namespace.
This namespace has one special outlier: the `ImmutableArray<T>` is a simple struct which wraps an array, but does not allow setting elements at a given index.
As such, it's just as fast as using an array - the fastest collection - while granting all the benefits of immutability.
But as an outlier, this collection also has downsides: adding or removing any elements runs in `O(n)` because you need to make a new array with the changes, and copy existing contents.
Note that this is perfectly fine for small collections, as copying arrays is a common operation that's very optimized on most CPUs.

> 💡 Always use `ImmutableArray<T>` whenever you don't need fast insertion and removal!

Contrary to common intuition, "persistent" does not refer to writing to a disk or database. 
Instead, it refers to the fact that modifications to the collection return a new one, therefore the original collection is persisted.
You can emulate this behavior with mutable collections, but then every modifcation would first involve an `O(n)` copy.
Persistent data structures are those that allow persistent modifications in `O(log n)` or even `O(1)` time.

This is great for some use-cases, e.g. when traversing a directory tree recursively and you want to remember the path you have taken.
With regular collections, you are limited to a depth-first search. 
I dare you, try to implement a BFS through a directory tree while remembering the path using only mutable collections. No copying.

Persistent collections are great in principle, because they are immutable. 
With them, you get all the benefits of immutability: 
These collections are thread-safe and can be safely be shared. 
You can avoid defensive copying of collections in many cases.
It's much easier to reason about code without mutation.

But as a downside, to allow these fast immutable mutations, these collections have some trade-offs on a lower level.
For example, most mutable collections in C# are backed by an array. 
A consecutive, cache-friendly block of memory, that the JIT compiler can use to implement some effective optimizations.
Besides cache-friendliness and fast iteration times, arrays are also fairly cheap to copy all at once (a single `memcpy` call), and they cause less memory fragmentation.

In contrast, all immutable collections (with `ImmutableArray<T>` being the exception) use a system of individually allocated nodes and pointers in some sense.
While the property of fast immutable modifications can be beneficial, this implementation leads to fairly large constants.
This is because heap allocation is very expensive compared to most other types of operations. 
And when you need to wrap every element in a node on the heap, the overhead quickly piles up.
Not only do you get the allocation overhead, but also increased memory fragmentation, and additional pointer indirections that need to be resolved by the CPU.

> ⚠️ Only use immutable collections if you really need fast persistent updates. Otherwise prefer using `IReadOnly` interfaces or `ImmutableArray<T>`!

TODO REST






# Someone on the internet is wrong! I need to correct him!

People make mistakes, and so do I. If you have a suggestion, you can contact me via email or [leave me an issue on GitHub](https://github.com/XDracam/DracTec-Dev-Blog/issues). Thanks!