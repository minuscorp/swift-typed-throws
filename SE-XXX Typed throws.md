Hello Swift Community. I started a conversation about this on the [Evolution|Proposals](https://forums.swift.org/t/typed-throw-functions/38860/) Forum and some people found interesting that might be worth to write a pitch about it.

- Typed `throws`
- Proposal: SE-NNNNN
- Authors: @minuscorp
- Review Manager: TBD
- Status: Proposed

# Introduction

Swift Error handling system seems lacking of the other features that the Swift language has: a type system. Having typed thrown errors allow users to make a more faine-grained error handling and a more safety when facing error handling procedures.


# Motivation

Many Swift APIs provide methods marked as `throws`, this, however, provides little to no information to the consumer about what kind of error is being thrown, if any, unless the provider adds documentation about it, which means nothing to the compiler neither the safety of the code being produced. Documentation tends to get stale and then, chances of producing unexpected errors for the user are higher. We illustrate this kind of current behaviour in the following way:

```swift
import ExternalLibrary

do {
    let newNumberOfFiles = try ExternalUtility.incrementNumberOfFilesOnDirectory(at: path)
} catch let error as NSError { // Maybe throws an NSError because it uses `FileManager`
    dump(error)
    recoverFromNSError()
} catch let error as? ExternalError { // Maybe throws an Error from the own library which is defined.
    dump(error)
    recoverFromLibraryError()
} catch { // No idea of what might be going on here
    fatalError()
    // or just let it go ahead with the error or stop the execution for `error` reason.
}
```

What this proposal is about is giving resilience and type safety to an area of the Swift language that lacks of it, ganing both in safety for the developer not just in type system but in terms of reducing the number of possible path error recovering that the developer might have to face when consuming an API.
The proposed semantics are pretty simple and additive from which we have today:

```
function-signature ::= params-type params-type throws? throws-type?
```

In the snippet below, we try to ilustrate which would be the end result of the implementation of the current proposal:

```swift
import ExternalLibrary

// Given: 
public func incrementNumberOfFilesOnDirectory(at path: String) throws ExternalError -> Int { ... }

enum ExternalError: Error {
    case pathNotFoundOrValid
    case maximumNumberOfFilesReached
}

// Then:
do {
    let newNumberOfFiles = try ExternalUtility.incrementNumberOfFilesOnDirectory(at: path)
} catch { // Type-safe: error is ExternalError
    dump(error)
    recoverFromLibraryError(error)
}

```

Scenarios where we can make use of the Swift compiler with the current proposal:

```swift
public func foo() throws MyError {
    throw OtherError.baz // error: OtherError cannot be casted to MyError.
}

public func foo() throws MyError {
  do {
    try typedThrow() // Throws OtherError inside a do statement there're no restrictions about the throwing type.
  } catch {
    throw MyError.baz 
  }
}

public func foo() throws MyError {
  do {
    try typedThrow1() // Throws OtherError
    try typedThrow2() // Throws AnotherError
  } catch { // compiler cannot ensure which of the two errors are being emmited
    dump(error) // Is casted to `Error`
  }
}
```

And avoid mistyped catching clauses:

```swift
do {
   try typedThrow() // Throws MyError
} catch let error as NSError { // error: NSError cannot be casted to MyError
   fatalError() 
}
```

And where we avoid dead code:

```swift
// bad 1
do { 
    try untypedThrow() 
} catch let error as TestError {
  // error is `TestError`
} catch { /* dead code */ }

// bad 2
do { try untypedThrow() }
catch {
  let error = error as! TestError
  // error is `TestError`
}

// good
do { try typedThrow() }
catch {
  // error is `TestError`
}
```

Also, there's no impact over `rethrows` clause, as he can inherit from its inner throwing type:

```swift
func foo<T>(_ block: () throws T -> Void) rethrows T
```

In the example above there's no need to constraint `T: Error`, as other any kind of object that does not implement `Error` will throw a compilation error, but it is handy to match the inner `Error` with the outer one.


In terms of consistency, there's a inequality in terms of how Swift type errors. Swift introduced `Result` in its 5.0 version as a way of somehow fill the gap of the situation of success or failure in an operation. This impact in the code being wirtten, making `throws` a second class tool for error handling. This idea leads to code being written in the following way:

With `Result` you could always:

```swift
func getCatResult() -> Result<Cat, CatError>

// You'll never know that the error being thrown is CatError
func getCatResult() throws -> Cat
```

Which is totally valid and correct, but declares a clear disadvantage against people using `throws`. One has the safety of unwrapping the correct `Error` type but the other one doesn't. With this proposal we balance both use cases like so:

```swift
func getCatResult() throws CatResult -> Cat
```

Would be totally valid. So you can put face to face `Result` and `throws` and perform the same operations with different semantics (and the semantics depend on the developer needs) in a free way, and not being obliged to choose one solution over the other one just because it cannot reach the same goal with it.

# Source compatibility
This change is purely additive and should not affect source compatibility.

# Effect on ABI stability
No known effect.

# Effect on API resilience
No known effect.
