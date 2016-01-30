# Feature name

* Proposal: Tuple Conversion and Composition [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Swift Developer](https://github.com/swiftdev)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

This proposal introduces a way to concatenate tuple types together:

```swift
typealias ABCD = (A,B)+(C,D) // Same as (A,B,C,D)
```

It also allows tuples to be converted to other tuples that are the same when flattened:

```swift
(a,(b,c),d) as ((a,b),(c,d))
```

This proposal aims to make the minimal changes possible to make tuples as powerful as possible.

<!-- 
> A short description of what the feature is. Try to keep it to a
> single-paragraph "elevator pitch" so the reader understands what
> problem this proposal is addressing.
>
> Swift-evolution thread: [link to the discussion thread for that proposal](https://lists.swift.org/pipermail/swift-evolution)
-->

## Motivation

<!-- 
> Describe the problems that this proposal seeks to address. If the
> problem is that some common pattern is currently hard to express, show
> how one can currently get a similar effect and describe its
> drawbacks. If it's completely new functionality that cannot be
> emulated, motivate why this new functionality would help Swift
> developers create better Swift code.
-->

This makes the type system much more powerful, and allows APIs that compose types to do so much more simply and elegantly.

This functionality will allow for greater code reuse, but is primarily for libraries to be able to make more flexible composable and concise interfaces.

With these addition functions Swift can now define things like this:

### Examples

#### Joining two tuples together

```swift
let a = (1,2), b = (3,4)
let c = join(a, b) // (Int,Int,Int,Int)
func join(lhs: A, rhs: B) -> A + B {
    return (lhs, rhs) as (A + B)
}
```

#### Stack like operations on tuples

```swift
let (abcde)  = (1,2,3,4,5)
let (abcd,e) = pop(abcde)
let (abc,d)  = pop(abcd)
let (abce)   = push(abc, e)
assert(abce == (1,2,3,5))

func pop<A,B>(_ tuple: A + (B)) -> (A,B) {
  return tuple as (A,B)
}
func push<A,B>(_ tuple: A, value: B) -> A + (B) {
  return (tuple,value) as (A + B)
}
```

#### Composable ZipSequence

```swift
let a = [1,2,3]
let b = ["a", "b", "c"]
let c = [1.2, 2.3, 3.4]
let d = [nil, nil, 123]
let abc = a.zip(b).zip(c).zip(d) // AnySequence<(Int,String,Float,Int?)>

extension SequenceType {
    func zip<S: SequenceType>(sequence: S)
        -> AnySequence<Generator.Element + S.Generator.Element>
    {
        typealias E = Generator.Element + R.Generator.Element
        return AnySequence { () -> AnyGenerator<E> in
            var lgen = lhs.generate(), rgen = rhs.generate()
            return AnyGenerator { () -> E?
                if let l = lgen.next(), r = rgen.next() {
                    return (l, r) as E
                }
                else {
                    return nil
                }
            }
        }
    }
}
```

#### Split an arbitrary function, by type alone, so it can be partially applied

```swift
func partial<A,B,C>(f: A + B -> C) -> (A -> B -> C) {
    return { (a: A) in
        return { (b: B) in
            return f((a,b) as (A + B))
        }
    }
}
```

#### Reorder the parameters of arbitrary functions

```swift
func reorder<A,B,C,D>(f: A + B + C -> D) -> (B + A + C -> D) {
    return { (params: B + A + C) in
        let (b,a,c) = params as (B,A,C)
        return f( (a,b,c) )
    }
}
func reorder<A,B,C,D>(f: A + B + C -> D) -> (A + C + B -> D) {
    return { (params: A + C + B) in
        let (a,c,b) = params as (A,C,B)
        return f( (a,b,c) )
    }
}
```

#### Turn any function with a callback into one with a promise

```swift
func promise<A,B,C>(f: (A + (B->Void)) -> C) -> A -> (C, Promise<B>) {
    return { (a: A) in
        let promise = Promise<B>()
        let c: C = f(a) { (b: B) in
            promise.send(b)
        }
        return (c, promise)
    }
}
```

## Proposed solution

<!-- 
> Describe your solution to the problem. Provide examples and describe
> how they work. Show how your solution is better than current
> workarounds: is a cleaner, safer, or more efficient?
-->

This proposal adds a concatenate operator to Swift tuple types, it can be used in type expressions. I'm suggesting `+` for the operator:

```swift
typealias U = (A,B,C) + (D,E,F)
```

This produces a tuple type concatenating the types of those two tuples:

```swift
typealias U = (A,B,C,D,E,F)
```

This proposal also extends the `as` operator to be able to convert between tuples with the same elements, when flattened. For example:

```swift
let x = (a,b,(c,(d,e))) as (a,b,c,d,e)
let y = x as ((a,b),c,(d,e))
```

## Detailed design

<!-- 
> Describe the design of the solution in detail. If it involves new
> syntax in the language, show the additions and changes to the Swift
> grammar. If it's a new API, show the full API and its documentation
> comments detailing what it does. The detail in this section should be
> sufficient for someone who is *not* one of the authors to be able to
> reasonably implement the feature.
-->

Single element tuples are not unwrapped before composed:

```swift
typealias T = (Int,Int)
typealias U = T + (Int) + (T)
```

Is equivalent to:
```swift
typealias T = (Int,Int)
typealias U = (Int,Int,Int,T)
```

The `+` operator only works on types that are available at compile time, so this will not work:

```swift
self.dynamicType + (Int)
```

## Impact on existing code

No impact, it only adds functionality. It should allow a few things in the standard library to be simplified and generalized though. If standard library simplifications are done as the same body of work it will require a small migration for any of the interfaces that change.

<!-- 
> Describe the impact this change will have on existing code. Will some
> Swift applications stop compiling due to this change? Will applications still
> compile but produce different behavior than they used to? Is it
> possible to migrate existing Swift code to use a new feature or API
> automatically?
-->

## Alternatives considered

This solution was found iteratively. Other operators instead of + were considered. It was chosen as they were unambiguous, simple, and concise.

A tuple could have a type per-index, but this would require a lot of other changes to code, and would not help this functionality be implementation. It would also get complex if you wanted an index on a sub-tuple to work.

The proposed solution can be generalized when the size of the tuple is not known, indices cannot be generalized so easily.

It was also considered, instead of the `as` operator, to have the following functions:

```swift
func prefix<A,B>(_ tuple: A + B) -> A { ... }
func suffix<A,B>(_ tuple: A + B) -> B { ... }
func concat<A,B>(lhs: A, rhs: B) -> A + B { ... }
```

This is more explicit, but not as concise, and would not be able to be defined in pure-swift anyway. The `as` operator already implies it's the compiler doing the conversion, not some function, so it's more obvious how it is working.

It was also considered whether the + operator is necessary, you could do this instead:
```swift
    (A) (B)
```

However that will probably lead to mistakes where something is left out.

<!-- 
> Describe alternative approaches to addressing the same problem, and
> why you chose this approach instead.
-->
