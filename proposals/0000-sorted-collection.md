# Sorted Collections

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Letanyan Arumugam](https://github.com/Letanyan), [Karoy Lorentey](https://github.com/lorentey)
* Review Manager: TBD
* Status: **Awaiting implementation**

[//]: # "*During the review process, add the following fields as needed:*"

[//]: # "* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)"
[//]: # "* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)"
[//]: # "* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)"
[//]: # "* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)"
[//]: # "* Previous Proposal: [SE-XXXX](XXXX-filename.md)"

## Introduction

Sorted collections allow one to maintain a collection of elements which are stored and accessed in a user-controlled order. Storing elements in a defined order can provide better efficiency for certain operations, such as lookup, while always maintaining a defined order.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

### Performance

Sorted collections can implement **O(log n)** time lookups since they can perform a binary search on their contents.

A collection like a `SortedMultiset` can provide **O(log n)** insertions/removals, while an array could only provide **O(n)** insertions/removals into an arbitrary location.

Set like operations such as union/merge and intersection could be done with **O(log n)** time with a `SortedSet` or `SortedDictionary`. This complexity is in contrast to `Dictionary` and `Set` which would require **O(n)** time.

### Use Cases

It is useful to have a way to represent collections of data with a user-controlled order. Structures such as priority queues, sorted dictionaries and sets have helpful features. Namely displaying end-user facing results, job scheduling, pathfinding, encoding and others.

`Set` and `Dictionary` do not define order to their iteration, it would be useful to have structures that can behave like `Set` and `Dictionary` but have an order.

## Proposed solution

We propose introducing three concrete sorted collections namely `SortedMultiset`, `SortedSet` and `SortedDictionary`. We will also include two protocols to generalise the concept. Types in the standard library shall also conform to said protocols where appropriate.

## Detailed design

### Sorted Collection Protocols:

```swift
protocol SortedCollection: Collection {
  associatedtype Key: Comparable
  associatedtype Keys: SortedCollection
    where Keys.Element == Key, Keys.Key == Key

  // Returns the index for the given key or `nil` of the key is not present.
  func index(for key: Key) -> Index?

  /// A sorted collection of all the keys for this collection.
  var keys: Keys { get }
}


protocol SortedInsertableCollection: SortedCollection {
  /// Returns the index of where the given key would be after insertion and
  /// whether a key already exists that is equal to the given key.
  ///
  /// - Returns: `keyAlreadyExists` must be return true if a key that is equal
  ///   to the given key already exists in the collection, otherwise return
  ///   false. `location` must return the index where the given key would be
  ///   inserted.
  func insertionPoint(
    for key: Key
  ) -> (keyAlreadyExists: Bool, location: Index)

  /// Inserts an element into the collection. The collection must remain sorted
  /// after the new element is inserted.
  ///
  /// - Returns: `inserted` must be false if the element was not inserted into
  ///   the collection, otherwise return true. `location` should return the
  ///   index of where the element was inserted or where it would have been
  ///   inserted.
  mutating func insert(
    _ element: Element
  ) -> (inserted: Bool, location: Index)
}

extension SortedCollection {
  /// Returns the element that is associated with `key`.
  subscript(key: Key) -> Element? { ... }
}
```

#### Default Implementations

```swift
extension SortedCollection where Keys == Self {
  var keys: Self { ... }
}

extension SortedCollection where Key == Element {
  func index(for key: Key) -> Index? { ... }
}

extension SortedInsertableCollection where Key == Element {
  func insertionPoint(
    for key: Key
  ) -> (keyAlreadyExists: Bool, location: Index) { ... }
}
```

### Sorted Multiset:

```swift
struct SortedMultiset<Element: Comparable>:
  BidirectionalCollection, ExpressibleByArrayLiteral,
  SortedInsertableCollection, Equatable, CustomReflectable,
  CustomStringConvertible, CustomDebugStringConvertible { ... }

extension SortedMultiset: Hashable where Element: Hashable { ... }
extension SortedMultiset: Encodable where Element: Encodable { ... }
extension SortedMultiset: Decodable where Element: Decodable { ... }
```

#### Additional API:

```swift
extension SortedMultiset {
  /// Create a sorted multiset from a finite sequence of items. The sequence need
  /// not be sorted. If the sequence contains duplicate items, all of them are kept,
  /// in the same order.
  init<S: Sequence>(unsortedElements elements: S)
  where S.Element == Element { ... }

  /// Create a sorted multiset from a sorted finite sequence of items.
  /// If the sequence contains duplicate items, all of them are kept.
  init<S: Sequence>(sortedElements elements: S)
  where S.Element == Element { ... }

  /// Create an empty sorted multiset.
  init() { ... }
}

extension SortedMultiset {
  /// Inserts all the elements from another sorted multiset into this sorted
  /// multiset.
  mutating func insert(contentsOf other: SortedMultiset) { ... }

  /// Inserts all the elements from another sorted set into this sorted
  /// multiset.
  mutating func insert(contentsOf other: SortedSet<Element>) { ... }

  /// Inserts all the elements from a sequence into this sorted multiset.
  mutating func insert<S: Sequence>(contentsOf seq: S)
  where S.Element == Element { ... }
}

extension SortedMultiset {
  /// Removes and returns the first element of the multiset if it isn't empty.
  ///
  /// - Returns: The first element of the multiset if the multiset
  ///   is not empty; otherwise, `nil`.
  mutating func popFirst() -> Element? { ... }

  /// Remove the member referenced by the given index.
  @discardableResult
  mutating func remove(at index: Index) -> Element { ... }

  /// Remove the member at the given offset.
  @discardableResult
  mutating func remove(atOffset offset: Int) -> Element { ... }

  /// Remove all elements in the specified offset range.
  mutating func remove(atOffset offset: Range<Int>) { ... }

  /// Remove and return the first instance of `member` from the sorted multiset,
  /// or return `nil` if the sorted multiset contains no instances of `member`.
  @discardableResult
  mutating func remove(_ member: Element) -> Element? { ... }

  /// Remove a subrange from the multiset
  mutating func removeSubrange(_ range: Range<Index>) { ... }

  /// Remove a subrange from the multiset
  mutating func removeSubrange<R>(_ range: R)
  where R: RangeExpression, R.Bound == Index { ... }

  /// Remove all elements that return true for predicate.
  mutating func removeAll(where predicate: (Element) -> Bool) { ... }

  /// Remove all members from this sorted multiset.
  mutating func removeAll() { ... }
}
```

### SortedDictionary:

```swift
struct SortedDictionary<Key: Comparable, Value>:
  BidirectionalCollection, ExpressibleByDictionaryLiteral,
  SortedInsertableCollection, CustomReflectable,
  CustomStringConvertible, CustomDebugStringConvertible { ... }

extension SortedDictionary: Equatable where Value: Equatable { ... }
extension SortedDictionary: Hashable where Key: Hashable, Value: Hashable { ... }
extension SortedDictionary: Encodable where Key: Encodable, Value: Encodable { ... }
extension SortedDictionary: Decodable where Key: Decodable, Value: Decodable { ... }
```

#### Additional API:

```swift
extension SortedDictionary {
  /// Initialize a new sorted dictionary from an unsorted sequence of elements,
  /// using a stable sort algorithm.
  ///
  /// If the sequence contains elements with duplicate keys, only the last
  /// element is kept in the sorted dictionary.
  init<S: Sequence>(unsortedElements elements: S)
  where S.Element == Element { ... }

  /// Initialize a new sorted dictionary from a sorted sequence of elements.
  ///
  /// If the sequence contains elements with duplicate keys, only the last
  /// element is kept in the sorted dictionary.
  init<S: Sequence>(sortedElements elements: S)
  where S.Element == Element { ... }

  /// Initialize a new empty sorted dictionary.
  init() { ... }
}

extension SortedDictionary {
  /// Returns a new sorted dictionary containing the keys of this sorted
  /// dictionary with the values transformed by the given closure.
  ///
  /// - Parameter transform: A closure that transforms a value. `transform`
  ///   accepts each value of the sorted dictionary as its parameter and returns
  ///   a transformed value of the same or of a different type.
  /// - Returns: A sorted dictionary containing the keys and transformed values
  ///   of this dictionary.
  func mapValues<T>(
    _ transform: (Value) throws -> T
  ) rethrows -> SortedDictionary<Key, T> { ... }
}

extension SortedDictionary {
  /// A collection containing just the keys in this sorted dictionary.
  var keys: Keys { ... }

  /// A collection containing just the values in this sorted dictionary, in
  /// order of ascending keys.
  var values: LazyMapCollection<SortedDictionary<Key, Value>, Value> { ... }

  /// Provides access to the value for a given key. Nonexistent values are
  /// represented as `nil`.
  subscript(key: Key) -> Value? {
    get { ... }
    set(value) { ... }
  }

  /// Update the value stored in the sorted dictionary for the given key, or, if
  /// they key does not exist, add a new key-value pair to the sorted
  /// dictionary. Returns the value that was replaced, or `nil` if a new
  /// key-value pair was added.
  @discardableResult
  mutating func updateValue(_ value: Value, forKey key: Key) -> Value? { ... }
}

extension SortedDictionary {
  /// Combines elements from `self` with those in `other`. If a key is included
  /// in both sorted dictionaries, then strategy is used to decide which value
  /// to use.
  mutating func merge(
    _ other: SortedDictionary<Key, Value>,
    uniquingKeysWith strategy: (Value, Value) -> Value
  ) { ... }

  /// Combines elements from `self` with those in `other`. If a key is included
  /// in both sorted dictionaries, then strategy is used to decide which value
  /// to use.
  mutating func merge<S: Sequence>(
    _ other: S,
    uniquingKeysWith strategy: (Value, Value) -> Value
  ) where S.Element == (Key, Value) { ... }

  /// Return a sorted dictionary that combines elements from `self` with those
  /// in `other`. If a key is included in both sorted dictionaries, then
  /// strategy is used to decide which value to use.
  func merging(
    _ other: SortedDictionary<Key, Value>,
    uniquingKeysWith strategy: (Value, Value) -> Value
  ) -> SortedDictionary { ... }

  /// Return a sorted dictionary that combines elements from `self` with those
  /// in `other`. If a key is included in both sorted dictionaries, then
  /// strategy is used to decide which value to use.
  func merging<S: Sequence>(
    _ other: S,
    uniquingKeysWith strategy: (Value, Value) -> Value
  ) -> SortedDictionary where S.Element == (Key, Value) { ... }
}

extension SortedDictionary {
  /// Removes and returns the first key-value pair of the dictionary if the
  /// dictionary isn't empty.
  ///
  /// - Returns: The first key-value pair of the dictionary if the dictionary
  ///   is not empty; otherwise, `nil`.
  mutating func popFirst() -> Element? { ... }

  /// Remove and return the largest member in this sorted array, or return `nil`
  /// if the sorted array is empty.
  @discardableResult
  mutating func popLast() -> Element? { ... }

  /// Remove the key-value pair at `index` from this sorted dictionary.
  ///
  /// This method invalidates all existing indexes into `self`.
  @discardableResult
  mutating func remove(at index: Index) -> (key: Key, value: Value) { ... }

  /// Remove and return the (key, value) pair at the specified offset from the
  /// start of the sorted dictionary.
  @discardableResult
  mutating func remove(atOffset offset: Int) -> Element { ... }

  /// Remove all (key, value) pairs in the specified offset range.
  mutating func remove(atOffset offset: Range<Int>) { ... }

  /// Remove a given key and the associated value from this sorted dictionary.
  /// Returns the value that was removed, or `nil` if the key was not present in
  /// the sorted dictionary.
  @discardableResult
  mutating func removeValue(forKey key: Key) -> Value? { ... }

  /// Remove a subrange from the dictionary
  mutating func removeSubrange(_ range: Range<Index>) { ... }

  /// Remove a subrange from the dictionary
  mutating func removeSubrange<R>(_ range: R)
  where R: RangeExpression, R.Bound == Index { ... }

  /// Remove all elements that return true for predicate.
  mutating func removeAll(where predicate: (Element) -> Bool) { ... }

  /// Remove all elements from this sorted dictionary.
  mutating func removeAll() { ... }
}

extension SortedDictionary {
  /// Returns a new sorted dictionary containing the key-value pairs of the
  /// sorted dictionary that satisfy the given predicate.
  ///
  /// - Parameter isIncluded: A closure that takes a key-value pair as its
  ///   argument and returns a Boolean value indicating whether the pair
  ///   should be included in the returned dictionary.
  /// - Returns: A sorted dictionary of the key-value pairs that `isIncluded`
  ///   allows.
  func filter(
    _ isIncluded: (Element) throws -> Bool
  ) rethrows -> SortedDictionary<Key, Value> { ... }
}
```

### SortedSet:

```swift
struct SortedSet<Element: Comprable>:
  ExpressibleByArrayLiteral, SortedInsertableCollection, BidirectionalCollection,
  Equatable, CustomStringConvertible, CustomDebugStringConvertible,
  CustomReflectable { ... }

extension SortedSet: Hashable where Element: Hashable { ... }
extension SortedSet: Encodable where Element: Encodable { ... }
extension SortedSet: Decodable where Element: Decodable { ... }
```

#### Additional API:

```swift
extension SortedSet {
  /// Create a set from a finite sequence of items. The sequence need not be
  /// sorted. If the sequence contains duplicate items, only the last instance
  /// will be kept in the set.
  public init<S: Sequence>(unsortedElements elements: S)
  where S.Element == Element { ... }

  /// Create a set from a sorted finite sequence of items. If the sequence
  /// contains duplicate items, only the last instance will be kept in the set.
  public init<S: Sequence>(sortedElements elements: S)
  where S.Element == Element { ... }

  /// Create an empty set.
  public init() { ... }
}

extension SortedSet {
  /// Inserts the given element into the set unconditionally.
  ///
  /// If an element equal to `newMember` is already contained in the set,
  /// `newMember` replaces the existing element.
  ///
  /// - Parameter newMember: An element to insert into the set.
  /// - Returns: The element equal to `newMember` that was originally in the
  ///   set, if exists; otherwise, `nil`. In some cases, the returned element
  ///   may be distinguishable from `newMember` by identity comparison or some
  ///   other means.
  @discardableResult
  public mutating func update(with newMember: Element) -> Element? { ... }
}

extension SortedSet {
  /// Removes and returns the first key-value pair of the dictionary if the
  /// dictionary isn't empty.
  ///
  /// - Returns: The first key-value pair of the dictionary if the dictionary
  ///   is not empty; otherwise, `nil`.
  public mutating func popFirst() -> Element? { ... }

  /// Remove and return the largest member in this set, or return `nil` if the
  /// set is empty.
  @discardableResult
  public mutating func popLast() -> Element? { ... }

  /// Remove the member referenced by the given index.
  @discardableResult
  public mutating func remove(at index: Index) -> Element { ... }

  /// Remove the member at the given offset.
  @discardableResult
  public mutating func remove(atOffset offset: Int) -> Element { ... }

  /// Remove all elements in the specified offset range.
  public mutating func remove(atOffset offset: Range<Int>) { ... }

  /// Remove the member from the set and return it if it was present.
  @discardableResult
  public mutating func remove(_ member: Element) -> Element? { ... }

  /// Remove a subrange from the set
  @inlinable
  public mutating func removeSubrange(_ range: Range<Index>) { ... }

  /// Remove a subrange from the set
  public mutating func removeSubrange<R>(_ range: R)
  where R: RangeExpression, R.Bound == Index { ... }

  /// Remove all elements that return true for predicate.
  public mutating func removeAll(where predicate: (Element) -> Bool) { ... }

  /// Remove all members from this set.
  public mutating func removeAll() { ... }
}
```

### Standard Library Conforming Types

```swift
extension Range: SortedCollection
where Bound: Strideable, Bound.Stride : SignedInteger {...}

extension Slice: SortedCollection where Base: SortedCollection { ... }

extension LazyFilterCollection: SortedCollection
where Base: SortedCollection { ... }
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

### Use closures to determine order and/or equality

While at first it might seem more convenient to have an easy way to customise sorting of these various collections they are a couple sacrifices that one must make:

- Not being able to check if closures are equal would mean that operations between different sorted collections could not be as performant as possible.
- Operations that use multiple sorted collections would need to disambiguate between which collections closure to use. This would mean a type like `SortedSet` could not conform to `SetAlgebra`.
- Since closures aren't `Codable` it will not be possible to have the collections be `Codable`.
