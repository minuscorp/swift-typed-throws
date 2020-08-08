# Typed throws

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Jorge (@minuscorp)](https://github.com/minuscorp), [Torsten Lehmann](https://github.com/torstenlehmann)
* Review Manager: TBD
* Status: **Proposed**

## Introduction

`throws` in Swift is missing the possibility to use it with specific error types. On the contrary [`Result`](https://developer.apple.com/documentation/swift/result) and [`Future`](https://developer.apple.com/documentation/combine/future) support specific error types. This is inconsistent without reason. The proposal is about introducing the possibility to support specific error types with `throws`.

Swift-evolution thread: [Typed throw functions - Evolution / Discussion - Swift Forums](https://forums.swift.org/t/typed-throw-functions/38860)

## Motivation

Swift is known for being explicit about semantics and using types to communicate constraints that apply to specific structures and APIs. Some developers are not satisfied with the current state of `throws` as it is not explicit about errors that are thrown. These leads to the following issues with `throws` current behaviour.

### Communicates less information than `Result` or `Future`

Assume you have this Error type

```swift
enum CatError: Error {
    case sleeps
    case sitsAtATree
}
```

Compare

```swift
func callCat() -> Result<Cat, CatError>
```

or

```swift
func callFutureCat() -> Future<Cat, CatError>
```

with

```swift
func callCatOrThrow() throws -> Cat
```

`throws` communicates less information about why the cat is not about to come to you.

### Inconsistent explicitness compared to `Result` or `Future`

`throws` it's not consistent in the order of explicitness in comparison to `Result` or `Future`, which makes it hard to convert between these types or compose them easily.

```swift
func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCatOrThrow())
    } catch {
        // won't compile, because error type guarantee is missing in the first place
        return Result.failure(error)
    }
}
```

```swift
func callAndFeedCat2() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCatOrThrow())
    } catch let error as CatError {
        // compiles
        return Result.failure(error)
    } catch {
        // won't compile, because exhaustiveness can't be checked by the compiler
        // so what should we return here?
        return Result.failure(error)
    }
}
```

### Code is not self documenting

Do you at least once stopped at a throwing (or loosely error typed) function wanting to know, what it can throw? Here is a more complex example. Be aware that it's not about a throwing function, but the problem applies to throwing functions as well. The root issue is the loosely typed error.

[urlSession%28_:task:didCompleteWithError:%29 | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsessiontaskdelegate/1411610-urlsession)

```swift
optional func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?)
```

> The only errors your delegate receives through the error parameter are client-side errors, such as being unable to resolve the hostname or connect to the host.

Ok so we show a pop-up if such an error occurs like we would if we have a response error. Furthermore we want to provide task cancellation because it's a big file download.

Now the user cancels the file download and sees the error pop-up which is **not** what we wanted.

What went wrong? You hopefully read the documentation of [cancel() | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsessiontask/1411591-cancel) or you debug the process and are suprised of this unexpected `NSURLErrorCancelled` error.

Now you know (at least) that you want to ignore this specific error. But which other errors you are not aware of?

### Outdated API documentation

API documentation could become outdated, because it's not checked by the compiler.

Assume some scarce documentation (more thorough documentation is even more likely to get outdated).

```swift
/// throws CatError
func callCatOrThrow() throws -> Cat
```

Let's update the method to load this cat from the network:

```swift
/// throws CatError
func callCatOrThrow() throws -> Cat { // now throws NetworkError additionally
    let catJSON = try loadCatJSON() // throws NetworkError
    // ...
}

struct NetworkError: Error {}
```

And there you have it. No one will check this `NetworkError` in specific catch clauses, even though it's not unlikely to have another error message for network issues.

### Potential drift between thrown errors and catch clauses

The example from section "Outdated API documentation" shows the issue where new errors from an updated API are not recognized by the API user. It's also possible that catched errors are replaced or removed by an updated API. So we end up with outdated catch clauses:

```swift
/// throws CatError, NetworkError
func callCatOrThrow() throws -> Cat
```
gets updated to

```swift
/// throws CatError, DatabaseError
func callCatOrThrow() throws -> Cat

struct DatabaseError: Error {}
```

No we have outdated catch clauses

```swift
do {
    let cat = try callCatOrThrow()
} catch let error as CatError {
    // CatError will be catched here
} catch let error as NetworkError {
    // won't happen anymore
} catch {
    // DatabaseError will be catched here
}
```

### `Result` is not the go to replacement for `throws` in imperative languages

Using explicit errors with `Result` has major implications for a code base. Because the exception handling mechanism ("goto catch") is not built into the language (like `throws`), you need to do that on your own, mixing the exception handling mechanism with domain logic (same issue we had with manual memory management in Objective-C before ARC).

#### Approach 1: Chaining Results

If you use `Result` in a functional (i.e. monadic) way, you need extensive use of `map`, `flatMap` and similar operators.

Example is taken from [Question/Idea: Improving explicit error handling in Swift (with enum operations) - Using Swift - Swift Forums](https://forums.swift.org/t/question-idea-improving-explicit-error-handling-in-swift-with-enum-operations/35335).

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

That's the functional way of writing exceptions, but Swift does not provide enough functional constructs to handle that comfortably (compare with [Haskell/do notation](https://en.wikibooks.org/wiki/Haskell/do_notation)).

#### Approach 2: Unwrap/switch/wrap on every chaining/mapping point

We can also just unwrap every result by switching over it and wrapping the value or error into a result again.

```swift
func userResultFromStrings(strings: [String]) -> Result<User, GenericError>  {
    let firstNameResult = stringResultFromArray(strings, at: 0, errorMessage: "Missing first name")
    
    switch firstNameResult {
    case .success(let firstName):
        let lastNameResult = stringResultFromArray(strings, at: 1, errorMessage: "Missing last name")
        
        switch lastNameResult {
        case .success(let lastName):
            return Result.success(User(firstName: firstName, lastName: lastName))
        case .failure(let genericError):
            return Result.failure(genericError)
        }
        
    case .failure(let genericError):
        return Result.failure(genericError)
    }
}
```

This is even more awful then the first approach, because now we are writing the implementation of the `flatMap` operator over an over again.

#### Approach 3: `throws` with specific error

So let's compare that to `throws`, if it would be usable with specific errors.

```swift
func stringFromArray(_ array: [String], at index: Int, errorMessage: String) throws GenericError -> String {
    guard array.indices.contains(index) else { throw GenericError(message: errorMessage) }
    return array[index]
}

func userResultFromStrings(strings: [String]) throws GenericError -> User  {
    let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
    let lastName = try stringFromArray(strings, at: 1, errorMessage: "Missing last name")
    return User(firstName: firstName, lastName: lastName)
}
```

The error handling mechanism is pushed aside and you can see the domain logic more clearly.

#### Error type conversions

In all 3 approaches we are omitting the issue of error type conversions which would be a topic for another proposal. But here's how it would look like for Approach 1 and 3 without further language constructs.

Example is taken from [Question/Idea: Improving explicit error handling in Swift (with enum operations) - Using Swift - Swift Forums](https://forums.swift.org/t/question-idea-improving-explicit-error-handling-in-swift-with-enum-operations/35335).

Approach 1:

```swift
struct FirstNameError: Swift.Error {}

func firstNameResultFromArray(_ array: [String]) -> Result<String, FirstNameError> {
    guard array.indices.contains(0) else { return Result.failure(FirstNameError()) }
    return Result.success(array[0])
}

func userResultFromStrings(strings: [String]) -> Result<User, GenericError>  {
    return firstNameResultFromArray(strings)
        .map { User(firstName: $0, lastName: "") }
        .mapError { _ in
            // Mapping from `FirstNameError` to a `GenericError`
            GenericError(message: "First name is missing")
        }
}
```

Approach 3:

```swift
func firstNameResultFromArray(_ array: [String]) throws FirstNameError -> String {
    guard array.indices.contains(0) else { throw FirstNameError() }
    return array[0]
}

func userResultFromStrings(strings: [String]) throws GenericError -> User  {
    do {
        let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
        return User(firstName: firstName, lastName: "")        
    } catch {
        // Mapping from `FirstNameError` to a `GenericError`
        throw GenericError(message: "First name is missing")
    }
}

```

### Multiple layers of error flow

`// TODO: Add examples and explain`

### Objective-C behaviour

In Objective-C, all current functions that take a double-pointer to `NSError` (a typical pattern in Foundation APIs) have an implicit type in the function signature, and, as has been pointed out, this does not provide more information that the current generic `Error`. But developers were free to subclass `NSError` and add it to its methods knowing that, the client would know at call time which kind of error would be raised if something went wrong.

The assumption that every error in the current Apple's ecosystem was an `NSError` an hence, convertible to `Error`, made `throws` loose its type. But nowadays, there are APIs (`Decodable` -> `DecodingError`, `StringUTF8Validation` -> `UTF8ValidationError`, among others) that makes the correct use of the Swift's throwing pattern but the client (when those APIs are available), cannot distinguish one from the other one.

The Swift Standard Library has left behind its own proper error handling system over the usage of `Optionals`, which are not meant to represent an `Error` but even the total erasure of it, leaving into a `nil` over the error being produced, leaving to the client no choice on how to proceed: unwrapping means that something wen't wrong, but I have any information about it. How does the developer should proceed.

Those methods can easily be `throws` with or without type, because the developer has already a tool to reduce the error to a `nil` value with `try?`, so, why limiting the developer to make a proper error handling when the tools are already there but we decice to just ignore them?


## Proposed solution
## Detailed design


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
}
    
do { try fooThrower() }
catch .bar { ... }
catch .baz { ... }
```

As `Foo` is the single type that can be thrown, we no longer need to specify the type either when throwing it nor when catching it.

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


### Error scenarios considered

#### Scenario 1: Specific thrown error, general catch clause

```swift
func callCat() throws CatError -> Cat

struct CatError {
    reason: String
}

do {
    let cat = try callCat()
} catch {
    // error is inferred as `CatError`
    // so this would compile
    let reason = error.reason
}
```

#### Scenario 2: Specific thrown error, specific catch clause

```swift
func callCat() throws CatError -> Cat

struct CatError {
    reason: String
}

do {
    let cat = try callCat()
} catch error as CatError { // ensure `CatError` even if the API changes in the future
    // error is inferred as `CatError`
    // so this would compile
    let reason = error.reason
}
```

No general catch clause needed. If there is one, compiler will show a warning or error.

#### Scenario 3: Specific thrown error, multiple catch clauses

```
func callCat() throws CatError -> Cat

enum CatError {
    case sleeps, sitsOnATree
}

do {
    let cat = try callCat()
} catch .sleeps {
    // handle error
} catch .sitsOnATree {
    // handle error
}
```

#### Scenario 4: Unspecific thrown error

- Current behaviour of Swift applies

### Type erasure

Erasing an error type of a function that throws is easy as

```
catch {
    // assume the error is inferred as `CatError`
    let typeErasedError: Error = error
}
```

### `rethrow` (generic errors in map, filter etc)

`// TODO: Explain, merge with already explained part`

### Error structure

Everything that applies to the error type of `Result` also applies to error type of `throws`. Mainly it needs to conform to `Swift.Error`.

### `async` and `throws`

`// TODO: Maybe we can remove that section`

### Equivalence between `throws` and `Result`

`// TODO: Explain motivation for another proposal and why we want to be compatible between throws and Result`

## Source compatibility

Being this change purely additive it would not affect on source compatibility.
Nevertheless, warnings might be produced in different scenarios Luke the examine below.

For instance, consider this function:

```swift
func fooThrower() throws
```

Are we allowed to change it to the following - while remaining source compatible?

```swift
func fooThrower() throws Foo
```

At first sight it shouldn't be a breaking change, since the equivalence between `Error` and `Foo` is evident.

But given the following client code:

```swift
do { try fooThrower() }
catch let error as? Foo { ... }
```
Which means that by changing to the latter function clients rebuilding will get a warning (saying that error is Foo).
So developers may have in consideration the addition of types to a plain throwing functions and deide whether the change will affect negatively in the client's code.

Lastly, but not least important, note the possible breaking effects of typing a throwing method into the client's code.

## Effect on ABI stability

No known effect.

## Effect on API resilience

No known effect.

## Alternatives considered

