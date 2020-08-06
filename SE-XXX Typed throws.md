Hello Swift Community. I started a conversation about this on the [Evolution|Proposals](https://forums.swift.org/t/typed-throw-functions/38860/) Forum and some people found interesting that might be worth to write a pitch about it.

- Typed `throws`
- Proposal: SE-NNNNN
- Authors: @minuscorp
- Review Manager: TBD
- Status: Proposed

# Introduction

Swift Error handling system seems lacking of the other features that the Swift language has: a type system. Having typed thrown errors allow users to make a more faine-grained error handling and a more safety when facing error handling procedures.


# Motivation

Many Swift APIs provide methods marked as `throws`, this, however, provides little to no information to the consumer unless the provider adds documentation about it, which means nothing to the compiler neither the safety of the code being produced, for example:

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

What is being proposed is much more safe and resilient, in terms of reducing possible paths that an application might take when recovering from an error:

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

Scenarios where you get type safety:

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
  } catch { // compiler cannot ensure which of the two errors are being emmited so it warns: Cannot infer throwing type from mixed type throwing methods.
    dump(error) // Is casted to `Error`
  }
}
```

Which can be autocorrected in:

```swift
do {
   try typedThrow1() // Throws OtherError
   try typedThrow2() // Throws AnotherError
} catch let error as OtherError { // The warning generates the missing catch clauses for the typed throws existant in the do block.
   dump(error)
} catch let error as AnotherError {
   dump(error)
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

Also, there's no impact over `rethrows`, as he can inherit from its inner throwing type:

```swift
func foo<T>(_ block: () throws T -> Void) rethrows T
```
In the example above there's no need to constraint `T: Error`, as other any kind of object that does not implement `Error` will throw a compilation error, but it is handy to match the inner `Error` with the outer one.

### Consistency with other APIs:

With `Result` you could always:

```swift
func getCatResult() -> Result<Cat, CatError>

// but never

func getCatResult() throws -> Cat
```

One has the safety of unwrapping the correct `Error` type but the other one doesn't, so with this proposal:

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
