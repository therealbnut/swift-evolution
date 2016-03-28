# Disambiguate Return Type With Void

* Proposal: [SE-NNNN](https://github.com/therealbnut/swift-evolution/blob/andrew-disambiguate-return-type/proposals/0000-disambiguate-return-type.md)
* Author(s): [Andrew Bennett](https://github.com/therealbnut)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

Swift can resolve functions based on the return type. However, when the result is unused a single function often can't be resolved without explicitly specifying the type.

    func example() { ... }
    func example() -> Int { ... }
    
    example() as Void
    example() as Int

This proposal disambiguates some cases:

 * Preferring functions with a **`Void`** return type when the result **is** discarded.
 * Preferring functions with a **non-`Void`** type when the result **is not** discarded.

These example will be unambiguous:

    example() // will prefer a `Void` function
    let x = example() // will prefer a non-`Void` function

Swift-evolution thread: [here](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160328/013559.html)

## Motivation

Some functions perform side-effects **and** return a result. It incurrs a cost to calculate the result whether it is used or not. Specialized versions of these functions can mean the result doesn't need to be calculated. However these specializations often result in ambiguity.

The calls to `example()` in the following code are ambiguous:

    func example() { ... }
    func example() -> Int { ... }
    
    example()
    let x = example()

To make the calls unambiguous Swift currently has a few options:

    example() as Void
    example() as Int
    let _: Void = example()
    let y: Int = example()

All of these options are unncessarily verbose. This is particularily evident in cases with long types:

    let someValue: UnsafeMutablePointer<String> = someOtherExample(someArguments)

## Proposed solution

After this proposal the following calls will resolve unambiguously.

    example() // calls the Void version
    let x = example() // cals the non-`Void` version

The code can be more concise, and less cluttered by extraneous annotations.

## Detailed design

The proposal makes the following changes:

 * If a result is discarded the call will resolve to a `Void` function if it can do so unambiguously.
 * If a result is not discarded the call will resolve to a non-`Void` function if it can do so unambiguously.
 * Otherwise, type inference will work as before.

### Examples

These function signatures are used for the examples in this section:

    func example1() { ... }
    func example1() -> Int { ... }

    func example2() { ... }
    func example2() -> Int { ... }
    func example2() -> Float { ... }
    
    func example3(x: Int) -> Int { ... }
    func example3(y: String) -> Void { ... }
    
    func example4<T>() -> T { ... }

The following table shows the type inference on these examples:

| **Function Call** | **Old Result Type** | **Result Type** |
|--------------------|---------------------|-----------------|
| `example1()` | *ambiguous* | `Void` |
| `let x = example1()` | *ambiguous* | `Int` |
| `example2()` | *ambiguous* | `Void` |
| `let x = example2()` | *ambiguous* | *ambiguous* |
| `let x: Int = example2()` | `Int` | `Int` |
| `example3("abc")` | `Void` | `Void` |
| `let x = example3("abc")` | `Void` | `Void` |
| `example3(123)` | `Int` | `Int` |
| `let x = example3(123)` | `Int` | `Int` |
| `{ example1() }` | *ambiguous* | *ambiguous* |
| `example4()` | *ambiguous* | `Void` |
| `let x = example4()` | *ambiguous* | *ambiguous* |
| `let x: Int = example4()` | `Int` | `Int` |

## Impact on existing code

This proposal does not break any existing code. However it will allow some existing code to be simplified.

## Alternatives considered

There are a few alternatives within Swift currently. As mentioned:

    example() as Void
    example() as Int
    let _: Void = example()
    let y: Int = example()

Alternatively, as mentioned, it may be possible to remove the Void function and incur the cost of calculating the result and discarding it. This assumes that the functions are otherwise equivalent.

For this to work the function must be marked as `@discardableResult`. However this is a specialized version of the problem.

In the case of `@discardableResult` the compiler could build two versions of the function, one with the result, and one without. In the Void version the compiler could perform optimisations to remove any unecessary calculations.

The compiler optimisations for `@discardableResult` would be useful regardless of this proposal.

This proposal is still useful to disambiguate cases where the specializations have different implementations.

This proposal is also useful to disambiguate cases where there are multiple non-void specializations.
