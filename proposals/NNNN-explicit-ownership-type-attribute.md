# Explicit Ownership Type Attribute

* Proposal: [SE-NNNN](https://github.com/therealbnut/swift-evolution/blob/therealbnut-explicit-ownership/proposals/NNNN-explicit-ownership-type-attribute.md)
* Author(s): [Andrew Bennett](https://github.com/therealbnut)
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

<!-- A short description of what the feature is. Try to keep it to a
single-paragraph "elevator pitch" so the reader understands what
problem this proposal is addressing. -->

This proposal introduces a way to to concisely restrict which references can be strong. This can be used to detect and avoid retain cycles. It does this with a new opt-in type attribute.

Swift-evolution thread: [Discussion thread topic for that proposal](http://news.gmane.org/gmane.comp.lang.swift.evolution)

## Motivation

<!-- Describe the problems that this proposal seeks to address. If the
problem is that some common pattern is currently hard to express, show
how one can currently get a similar effect and describe its
drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Swift
developers create better Swift code. -->

Currently it's very easy to make retain cycles. They're easy to miss, they create many traps for beginners, they're often time-consuming to diagnose and fix for non-beginners.

The following code is riddled with unintentional strong references, and failed attempts to fix them:

```swift
class Parent {
  var children: [Child] = []

  var delegate: ParentDelegate? // 1. should be weak

  func append(children: [Child]) {
    self.children.appendContentsOf(children)

    for child in children {
      child.parent = self // 2. parent shouldn't be strong
      child.addCallback { // 3. should have used [weak self]
        dispatch_async(dispatch_get_main_queue()) {
          [unowned self] in // 4. should have been weak or strong
          self.delegate?.inform(self.result)
        }
      }
    }
  }

  lazy var result: Int = { // 5. should use [unowned self]
    self.children.reduce(0) {
      [weak self] (total, child) in // 6. should have been strong
      guard let strongSelf = self else { return 0 }
      return total + child.work(strongSelf)
    }
  }()
}
```

## Proposed solution

<!-- Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient? -->

This proposal introduces an opt-in type attribute "`@owns`". The attribute defines a list of types that can be strongly referenced by the attributed type. Developers can use this to enforce ownership structures. The compiler can produce compile-time errors if those structures are violated.

```swift
@owns(TypeA, TypeB)
class NewType {
  ...
  func doWork(callback: @owns(TypeA,TypeB) (Int) -> Void)
}
```

An `@owns` attribute defines a list of types something can 'own'.

## Detailed design

<!-- Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature. -->

Topographically this forms a directed acyclic graph, naturally avoiding retain cycles. The list of types in owns form the ordering of that graph.

 If a type has an `@owns` attribute it must comply with these rules:

 * Ownership is inherited, a type can own anything the types it owns can.
 * All types it directly stores must also have an `@owns` attribute.
 * It must be able to own all reference types it directly stores.
 * It must be able to own all types owned by value types it directly stores.

If a type does not have an `@owns` attribute it can own any type, whether they are annotated or not.

### Example Structures
In these examples the notation `A → B` means that A can hold a reference to B.

#### A→B→C

```swift
@owns(B) class A { ... }
@owns(C) class B { ... }
@owns( ) class C { ... }
```

#### A→(B→D, C→D)
Note: D does not need to deny A because D denies B, which denies A, (because it only allows D).

```swift
@owns(B, D) class A { ... }
@owns(D)    class B { ... }
@owns(D)    class C { ... }
@owns( )    class D { ... }
```

#### ⟳A↔B⟲
Note: This is a "strongly connected component", this could allow retain cycles, it is an explicit and easily-identified choice by the programmer.

```swift
@owns(A, B) class A { ... }
@owns(B, A) class B { ... }
```

#### A
Note: Value types are implicitly owned, so can be ignored in the annotation.

```swift
@owns( ) class A { let x: B }
struct B { ... }
```

#### A -> T
Note: Annotated value types must state if they can store a reference type.

```swift
@owns(T) struct A<T> { ... }
```

#### A→C
If a reference stores another reference via a value-type it must be able to do so.

```swift
@owns(T) struct A<T> { ... }
@owns( ) class B { ... }

// Valid:
         class C1 { let x: A<B> }
@owns(B) class C2 { let x: A<B> }
// Invalid:
@owns( ) class C3 { let x: A<B> }
```

The case of `C1` can be avoided if a linter ensures that classes are annotated.

### Example Warnings and Errors
Typically references that contradict the ownership rules will result in a compile-time error or a warning.

This should be very efficient to determine at compile time, as the graph is defined by annotations, and verification of that graph is done at a local scope.

#### Retain Cycle
This would produce an error because this arrangement allows A to own B, and B to own A, but A cannot own A.

```swift
@owns(B) class A { ... }
Error: 'B' can own 'A' causing a retain cycle.
@owns(A) class B { ... }
```

#### Unexpected Property
This would produce a warning, because the reference type C is not listed in the allowed types.

```swift
@owns(B) class A {
  let x: C
Error: 'A' cannot hold references to 'C'.
}
@owns(B) 
class C {
}
```

#### Unannotated Type
A warning is produced if an annotated type tries to store an un-annotated type.

```swift
@owns(B) class A {
Warning: 'B' does not have any ownership annotations.
  let x: B
}
class B {
}
```

However, an un-annotated type is allowed to store an annotated type:

```swift
class A {
  let x: B
}
@owns() class B {
}
```

#### Disjoint Type
This would produce an error, because C is used by A, but there are no rules connecting the two.

```swift
@owns(B) 
class A {
  let x: C
Error: 'A' has explicit ownership and cannot own 'C'.
}
@owns(D)
class C {
}
```

#### Collapsing strongly connected components
Given the graph `A→B→C→D`:

```swift
@owns(B) class A { ... }
@owns(C) class B { ... }
@owns(D) class C { ... }
@owns( ) class D { ... }
```

If we make `A` and `D` a strongly connected component (`⟳A↔D⟲`):

```swift
@owns(A, B, D) class A { ... }
@owns(D, A   ) class D { ... }
```

Then it also collapses `B→C` into its strongly connected component. Using the previous error examples the error message would be something like:

```
Error: 'B' cannot own other instances of 'B'.
```

However, it would be better if the compiler detects the owner of `B` which creates the strongly connected component and puts the error in that context:

```
Error: Allowing 'A' to own itself causes 'B' to have ambiguous ownership.
```

### Objective-C

Classes defined in Objective-C could cause the checks to fail. Objective-C classes and protocols could get compatible annotations. Objective-C could also get similar verification, although similar to nullability annotations it's unlikely to be strongly enforced.

## Impact on existing code

<!-- Describe the impact that this change will have on existing code. Will some
Swift applications stop compiling due to this change? Will applications still
compile but produce different behavior than they used to? Is it
possible to migrate existing Swift code to use a new feature or API
automatically? -->

The Swift standard library should have ownership attributes applied. For example, GeneratorType will have this annotation:

```swift
@owns(Element)
public protocol GeneratorType { ... }
```

Otherwise, because of the inheritence rules, it should not impact existing code.

## Alternatives considered

<!-- Describe alternative approaches to addressing the same problem, and
why you chose this approach instead. -->

### Blacklist
A blacklist is much more unconstrained than is necessary, and requires knowledge of the parent. Knowledge of the parent doesn't work in circumstances like libraries.

### Whitelist + Blacklist
The combination of the two requires more keywords, or labels inside `@owns(...)`. For example `@owns(deny:,allow:)`, this can more easily cause strongly-connected-compoents, and makes it harder to create a well defined ordering.

### Property annotation
By using attribute `strong` on properties the topological ordering could be derived from this. It does not handle containers, nor does it allow closures to be given appropriate attributes.

### Ordering syntax like operator precedence
This was considered, but it became very verbose and clumsy when applied to lots of different types.

### Rust-style Ownership
Rust-style ownership was considered, it allows a lot more to be done statically, and probably produces much more robust memory management. However the rust approach is conceptually complex, from the rust-lang documentation (https://doc.rust-lang.org/book/ownership.html):

> However, this system does have a certain cost: learning curve. Many new users to Rust experience something we like to call ‘fighting with the borrow checker’ [...] more experienced Rust developers report that once they work with the rules of the ownership system for a period of time, they fight the borrow checker less and less.

Swift aims to be approachable from the start, it aims to be a pleasure to work with, not a fight. Predictability and robustness are desirable because they make it easier to use Swift. This proposal is conceptually simple, and is also opt-in, it is straightforward and helpful when used, and doesn't hinder when not used.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.
