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

Furthermore, many developers are being pushed on abandon `throws` methods for `Result` ones, where the error can be easily typed. This create a great unbalance, and sometimes, a poorly choose of the tools to use in order to code because of language limitations.

But using explicit errors with `Result` has major implications for a code base. Because the exception handling mechanism ("goto catch") is not built into the language (like `throws`), you need to do that on your own in a "`Result` chaining" (i.e. monadic) way with `flatMap` and similar operators, if you don't want to unwrap/switch/wrap on every chaining/mapping point. Leading to code like this:

```swift
struct GenericError: Swift.Error {
    let message: String
}

struct User {
    let firstName: String
    let lastName: String
}

func stringResultFromArray(_ array: [String], at index: Int, errorMessage: String) -> Result<String, GenericError> {
    guard array.indices.contains(index) else { return Result.failure(GenericError(message: errorMessage)) }
    return Result.success(array[index])
}

// no problem with converting errors, because it is always `GenericError`
func userResultFromStrings(strings: [String]) -> Result<User, GenericError>  {
    return stringResultFromArray(strings, at: 0, errorMessage: "Missing first name")
        .flatMap { firstName in
            stringResultFromArray(strings, at: 1, errorMessage: "Missing last name")
                .flatMap { lastName in
                    return Result.success(User(firstName: firstName, lastName: lastName))
            }
    }
}
```

Which ends up in a code hard to read, maintain and even further, be interprreted by the compiler. With the needed `flatMap` operators the compiler will need more and more context to infer the different types, ending in a over-boilerplated code just because you couldn't use `throws` with a type.

In Objective-C, all current functions that take a double-pointer to `NSError` (a typical pattern in Foundation APIs) have an implicit type in the function signature, and, as has been pointed out, this does not provide more information that the current generic `Error`. But developers were free to subclass `NSError` and add it to its methods knowing that, the client would know at call time which kind of error would be raised if something went wrong.

The assumption that every error in the current Apple's ecosystem was an `NSError` an hence, convertible to `Error`, made `throws` loose its type. But nowadays, there are APIs (`Decodable` -> `DecodingError`, `StringUTF8Validation` -> `UTF8ValidationError`, among others) that makes the correct use of the Swift's throwing pattern but the client (when those APIs are available), cannot distinguish one from the other one.

The Swift Standard Library has left behind its own proper error handling system over the usage of `Optionals`, which are not meant to represent an `Error` but even the total erasure of it, leaving into a `nil` over the error being produced, leaving to the client no choice on how to proceed: unwrapping means that something wen't wrong, but I have any information about it. How does the developer should proceed.

Those methods can easily be `throws` with or without type, because the developer has already a tool to reduce the error to a `nil` value with `try?`, so, why limiting the developer to make a proper error handling when the tools are already there but we decice to just ignore them?


# Proposal

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

public func foo() throws { 
  do {
    try typedThrow1() // Throws OtherError
    try typedThrow2() // Throws AnotherError
  } catch { // compiler cannot ensure which of the two errors are being emmited
    dump(error) // Is casted to `Error`
    throw Error // We throw the type-erased error as allowed by the function signature
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

Type inference would benefit also of typed `throws` like shown in the following examples:

```swift
enum Foo: Error { case bar, baz }

func fooThrower() throws Foo {
    guard someCondition else {
        throw .bar
    }

    guard someOtherCondition else {
        throw .baz
    }

    [...]
    
do { try fooThrower() }
catch .bar { ... }
catch .baz { ... }
}
```
As `Foo` is the single type that can be thrown, we no longer need to type the type either when throwing it nor when catching it.

And where we avoid dead code:

```swift
do { 
    try untypedThrow() 
} catch let error as TestError {
  // error is `TestError`
} catch { /* dead code */ }

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

In the example above there's no need to constraint `T: Error`, as other any kind of object that does not implement `Error` will throw a compilation error, but it is handy to match the inner `Error` with the outer one. So all the family of functions in the Standard Library (`map`, `flatMap`, `compactMap`, etc.) that now receive a `rethrows`, can be added with their error typed variants just modifying the signature, as for example:

```swift
// current one:
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]

// added one:
func map<T, E>(_ transform: (Element) throws E -> T) rethrows E -> [T]
```

Also, in this regard, given:

```swift
enum E: Error { case failure }

func f<T>(_: () throws T -> Void) { print(T.self) }

f({ throw E.failure }) // closure gets inferred to be () throws E -> Void so this will compile fine 
```

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

And we cannot forget that as `Result(catching: )` works, this should work too to bridge in the other way:

```swift
extension Result {
    func get() throws Failure -> Success
}
```

Would be totally valid. So you can put face to face `Result` and `throws` and perform the same operations with different semantics (and the semantics depend on the developer needs) in a free way, and not being obliged to choose one solution over the other one just because it cannot reach the same goal with it.

There have been many allusions to Library Evolution and how it would affect to this proposal. Out approach with non `@frozen enum`s is quite similar than what happens with switch cases:

```swift
enum NonFrozenEnum: Error { case cold, warm, hot }

func wheathersLike() throws NonFrozenEnum -> Weather

try { wheathersLike() } 
catch .cold { ... }
catch .warm { ... }
catch .hot { ... } // warning: all cases were catched but NonFrozenEnum might have additional unknown values.
// So if the warning is resolved:
catch { // error is still a NonFrozenEnum }
```

So it maintains backwards compatibility emiting a warning instead of an error. An error could be generated if this proposal doesn't need to keep backwards compatibility with other previous Swift versions.

In terms of subtyping, we're respecting the current model, so:

```swift
class BaseError: Error {}
class SubError: BaseError {}

let f1: () -> Void = { ... }
let f2: () throws SubError -> Void = f1 // Converting a non-throwing function to a throwing one is allowed.
let f3: () throws BaseError -> Void = f2 // This will also be allowed, but just with the BaseError features, it'll be casted down.
let f4: () throws -> Void = f3 // Erase the throwing type is allowed at any moment.
```


# Source compatibility
This change is purely additive and should not affect source compatibility.

# Effect on ABI stability
No known effect.

# Effect on API resilience
No known effect.
