# Sorted Collections

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Letanyan Arumugam](https://github.com/Letanyan)
* Review Manager: TBD
* Status: **Awaiting implementation**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

Many algorithm's and data structures rely on having sorted guarantees. However,
currently the Swift Standard Library does not provide a way to allow types to
provide this semantic guarantee.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

Having a common entry point to write algorithms that rely on sorted guarantees
would be desirable for allowing communication between multiple libraries and
code bases. The Standard Library may also want to provide types that would
require the semantics of this proposal later.

Commonality is only useful if the use cases that use the common entry point are
useful on their own.

### Performance

The common way of improving search performance is to use a binary search which
relies on having a random access collection of sorted elements. Other structures
have structural qualities that inherently allow similar **O(lg n)** performance.

A collection that is continuously updated to be sorted can be more efficient
than calling sort every time you need to use the collection.

### Semantics

It is useful to have a way to represent collections of data with inherit order.
Structures such as priority queues, sorted dictionaries and sets have useful
features. Namely display results or prioritizing information.

## Proposed solution

Introducing a protocol that provides the semantic guarantees of having a
collection always be sorted.

```
protocol SortedCollection : Collection {
    mutating func insert(_ element: Element)
}
```

Note that there is no requirement for an `Element` to be `Comparable`. Not
having this requirement allows any conformer to be able to use any way of
providing sorted guarantees.

The proposed solution will provide types to have their elements declare sorting
by their `Comparable` conformance or through a closure.

## Detailed design

```
/// A collection that guarantees that the conforming type always has its
/// elements in a given  sorted order.
///
/// For example, when elements are inserted into the collection the collection
/// will have them inserted in a order determined by some comparison.
///
///     var xs = SortedArray()
///     xs.insert(5)
///     xs.insert(10)
///     xs.insert(0)
///
/// The order that xs has its elements would be as follows, 0, 5, 10.
protocol SortedCollection : Collection {
    /// Inserts an element into correctly corresponding comparative position.
    ///
    /// This method must ensure that any element passed in must be placed in
    /// its correct position within the collection. This position can be based
    /// on the elements relation using its conformance to Comparable, if such 
    /// a conformance is available. Or through a closure or some other method
    /// that determines the relational order between two elements.
    mutating func insert(_ element: Element)
}
```

## Source compatibility

This is purely additive.

## Effect on ABI stability

No effect on ABI.

## Effect on API resilience

Some types in the Standard Library may adopt this protocol and as such removal
of this would cause certain algorithms that rely on using a SortedCollection
constraint to be modified.

## Alternatives considered

### Constrain Element to Comparable

This would force collections to force elements to conform to `Comparable` if
they weren't. Collections may also want to have custom sorting that only makes
sense for certain scenarios. Take for example a table row, the sorting method is
not inherit to the row, but rather situational. Types should be allowed to 
provide this flexibility without being restrained.

### Require a Sorting Closure

On the other side of requiring `Element` to conform to `Comparable` is a general
closure. Forcing conformers to fulfill this may also be undesirable when not
required as it forces the conformer to include added overhead to the type that
may not be required.