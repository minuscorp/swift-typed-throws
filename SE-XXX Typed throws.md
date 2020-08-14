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

### Communicates less error information than `Result` or `Future`

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

If you write your own module with multiple throwing functions, that depend on each other, it's even harder to track down, which error can be thrown from which function.

```swift
func callKids() throws -> Kids // maybe you still know that this can only by a `KidsError`
func callSpouse() throws -> Spouse // but a `Spouse` can fail in many different ways right?
func callCat() throws -> Cat // `CatError` I guess

func callFamily() throws -> Family {
	let kids = try callKids()
	let spouse = try callSpouse()
	let cat = try callCat()
	return Family(kids: kids, spouse: spouse, cat: cat)
}
```

As a user of `callFamily() throws` it gets a lot harder to understand, which errors can be thrown. Even if reading the functions implementation would be possible (which sometimes is not), than you are usally forced to read the whole implementation, collecting all uses of `try` and investigating the whole error flow dependencies of the sub program. Which almost nobody does, and the people that try often produce mistakes, because complexity quickly outgrows.

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

### Code is less self documenting

Do you at least once stopped at a throwing (or loosely error typed) function wanting to know, what it can throw? Here is a more complex example. Be aware that it's not about a throwing function, but the problem applies to throwing functions as well. The root issue is the loosely typed error.

[urlSession(\_:task:didCompleteWithError:) | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsessiontaskdelegate/1411610-urlsession)

```swift
optional func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?)
```

> The only errors your delegate receives through the error parameter are client-side errors, such as being unable to resolve the hostname or connect to the host.

Ok so we show a pop-up if such an error occurs like we would if we have a response error. Furthermore we want to provide task cancellation because it's a big file download.

Now the user cancels the file download and sees the error pop-up which is **not** what we wanted.

What went wrong? You hopefully read the documentation of [cancel() | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsessiontask/1411591-cancel) or you debug the process and are surprised of this unexpected [`NSURLErrorCancelled`](https://developer.apple.com/documentation/foundation/1508628-url_loading_system_error_codes/nsurlerrorcancelled) error.

Now you know (at least) that you want to ignore this specific error. But which other errors are you not aware of?

### Outdated API documentation

API documentation could and usually become outdated, because it's not checked by the compiler. The developer's task of documenting every method is more than enough than adding and maintaining the `- throws:` documentation. Which, do not help too much, due to the fact that following the [Apple Developer Markup Formatting Reference](https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_markup_formatting_ref/index.html#//apple_ref/doc/uid/TP40016497-CH2-SW1) there's no official way to link from the documentation to the current reference of the throwing type. This leads to an undesired dead end, searching in the library for ourselves for the throwing type that the documentation claims that it does.

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

The example from section "[Outdated API documentation](#outdated-api-documentation)" shows the issue where new errors from an updated API are not recognized by the API user. It's also possible that catched errors are replaced or removed by an updated API. So we end up with outdated catch clauses:

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

### Objective-C behaviour

`// TODO: Not sure if we need that much history :D, but we can refer to the specific errors Swift already uses in its Codable protocol`

In Objective-C, all current functions that take a double-pointer to `NSError` (a typical pattern in Foundation APIs) have an implicit type in the function signature, and, as has been pointed out, this does not provide more information that the current generic `Error`. But developers were free to subclass `NSError` and add it to its methods knowing that, the client would know at call time which kind of error would be raised if something went wrong.

`// TODO: we should not pretend what is correct or the right way, there are always pros and cons, we should just summarize them`

The assumption that every error in the current Apple's ecosystem was an `NSError` an hence, convertible to `Error`, made `throws` loose its type. But nowadays, there are APIs (`Decodable` -> `DecodingError`, `StringUTF8Validation` -> `UTF8ValidationError`, among others) that makes the correct use of the Swift's throwing pattern but the client (when those APIs are available), cannot distinguish one from the other one.

`// TODO: Optionals have there place, I think we should just say that the are sometimes not expressive enough and don't support "go to catch" semantics. But we can refer to Optionals having some comparable static information regarding the error structure (so throws has another execution semantic)`

The Swift Standard Library has left behind its own proper error handling system over the usage of `Optionals`, which are not meant to represent an `Error` but even the total erasure of it, leaving into a `nil` over the error being produced, leaving to the client no choice on how to proceed: unwrapping means that something wen't wrong, but I have any information about it. How does the developer should proceed.

`// TODO: That should be up to the API developer. An argument would be "because it easier to read for common cases"`

Those methods can easily be `throws` with or without type, because the developer has already a tool to reduce the error to a `nil` value with `try?`, so, why limiting the developer to make a proper error handling when the tools are already there but we decice to just ignore them?



## Proposed solution

`// TODO: What should we put in here and what not?`

> Describe your solution to the problem. Provide examples and describe how they work. Show how your solution is better than current workarounds: is it cleaner, safer, or more efficient?

In general we want to add the possibility to use `throws` with a specific error.

```swift
func callCat() throws CatError -> Cat
```

Here is how `throws` with specific error would reduce the issues mentioned in "[Motivation](#motivation)".

### Communicates the same amount of error information like `Result` or `Future`

Compare

```swift
func callCat() -> Result<Cat, CatError>
```

with

```swift
func callCat() throws CatError -> Cat
```

It now contains the same error information like `Result`.

### Consistent explicitness compared to `Result` or `Future`

`throws` is now consistent in the order of explicitness in comparison to `Result` or `Future`, which makes it easy to convert between these types.

```swift
func callCat() throws CatError -> Cat

func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCat())
    } catch {
        // would compile now, because error is inferred as `CatError`
        return Result.failure(error)
    }
}
```

```swift
func callAndFeedCat2() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCat())
    } catch let error as CatError {
        return Result.failure(error)
    } catch {
    	// this catch clause would become obsolete because the catch is already exhaustive
    }
}
```

### Code is more self documenting

```swift
struct RequestCatError: Error {
	case network, forbidden, notFound, internalServerError, unknown
}

func requestCat() throws RequestCatError -> Cat
```

It's now guaranteed which errors can happen.


### Drift between thrown errors and catch clauses can be stopped

You have an API with

```swift
func callCat() throws CatError -> Cat
```

and you make sure that you are only aware of a `CatError` by explictly catching it

```swift
do {
    let cat = try callCat()
} catch let error as CatError {
    // CatError will be catched here
}
```

Now the API gets updated to

```swift
func callCat() throws TurtleError -> Cat
```

which would result in non compiling code on your side

```swift
do {
    let cat = try callCat()
} catch let error as CatError { // would not compile, because you catch something that is not thrown
}
```

You can now update you catch clauses to make sure that you are aware of the new error cases and that you handle them properly.

### `throws` is made for imperative languages

Let's compare the two approaches from above to `throws`, if it would be usable with specific errors.

#### Approach 3: `throws` with specific error

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

Be aware that in all 3 approaches we are omitting the issue of error type conversions which would be a topic for another proposal. But here's how it would look like for Approach 1 and 3 without further language constructs.

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

`// TODO: I think we should put that to the detailed solution part now`

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

#### Scenario 2: Multiple specific thrown errors, general catch clauses

`// TODO: Maybe this decision has ugly consequences for the future, if we want to improve type inference for the error`

```swift
func callKids() throws KidsError -> Kids
func callCat() throws CatError -> Cat

do {
    let kids = try callKids()
    let cat = try callCat()
} catch {
    // `error` is just `Swift.Error`
}
```

This can be corrected by an autocompletion approach, giving the option to the developer to write down like:

```swift
catch error as KidsError { ... } // Autocompletion
catch error as CatError { ... } // Autocompletion
```

Another example is a variant of the example above where you have already catched specifically one of the errors thrown:

```swift
catch error as KidsError { ... }
catch { /* error is Error because the type cannot be inferred */ }
```

This scenarios, fill the gap about the boilerplate needed to write the needed code to catch different errors from a single `do` clause, where generating warnings or errors for autocompleting the missing `catch` cases would be too intrusive and even break source compatibility. With this autompletion rules, we scan the different errors being thrown and which ones are already catched in order to autocomplete the remaining cases only if the developer requires them.

#### Scenario 3: Specific thrown error, specific catch clause

```swift
func callCat() throws CatError -> Cat

struct CatError: Error {
    let reason: String
}

do {
    let cat = try callCat()
} catch error as CatError { // ensure `CatError` even if the API changes in the future
    // error is inferred as `CatError`
    // so this would compile
    let reason = error.reason
}
```

In case there are two

No general catch clause needed. If there is one, compiler will show a warning or error.

#### Scenario 4: Specific thrown error, multiple catch clauses

```swift
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

#### Scenario 5: Unspecific thrown error

- Current behaviour of Swift applies

### Error structure

Everything that applies to the error type of `Result` also applies to error type of `throws`. Mainly it needs to conform to `Swift.Error`.

### Equivalence between `throws` and `Result`

`// TODO: Explain motivation for another proposal and why we want to be compatible between throws and Result`



## Detailed design

`// TODO: Decide what belongs here and what belongs into other sections`

> Describe the design of the solution in detail. If it involves new syntax in the language, show the additions and changes to the Swift grammar. If it's a new API, show the full API and its documentation comments detailing what it does. The detail in this section should be sufficient for someone who is not one of the authors to be able to reasonably implement the feature.

`// TODO: Motivation part is over, we should only speak about the solution`

### Swift grammar additions/changes

What this proposal is about is giving resilience and type safety to an area of the Swift language that lacks of it, ganing both in safety for the developer not just in type system but in terms of reducing the number of possible path error recovering that the developer might have to face when consuming an API.

We define the proposed semantics represented as how the function would be mangled or just as a pseudocode:

```
function-signature ::= params-type params-type throws? throws-type?
```

Which can be translated into a more Swift-related grammar in the former way:

```swift
(func name | init)(params: params-type...) (throws throws-type?)? (-> return-type)?
```

Where, following our kitty example, we could write:

```swift
// From:
/// throws CatError
func callCatOrThrow() throws -> Cat

// To:
func callCatOrThrow() throws CatError -> Cat

// Being CatError:
struct CatError: Swift.Error {}
```

### The grammar rules

In the example above, despite being pretty simple, stablishes all the rules that applies to typed throwing functions:

1. Any function or init method that is marked as `throws` can declare which type will be thrown from the function body.
2. Just one type of error (if any), can be added to the function signature.
3. The error **must** conform to `Swift.Error`, by inheritance or by direct conformation and follow the same rules that apply to the `throw` statement.

Whith these three simple rules, we can elaborate a bit more in depth which are the implications of this:

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

The most evident scenario for a do block with a single thrown error inside is that it is no longer needed to catch a certain type in the catch clauses, due to the fact that the generic catch clause will become strongly typed with the error type specified in the throwing function's signature.

This single simple scenario opens up to a whole family of possible scenarios that should be mentioned for the sake of clarity.
In first place we will specify different examples where the Swift compiler interprets the new code and make decisions about the code in ways that it wasn't working before:

### Inheritance

A typed `throws` function cannot throw other type unrelated (by inheritance). If this happens, a compilation error will be generated as in the following example:

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
```

Whether the following example is totally valid, where the general `catch` statement casts the `Error` being thrown from the function to the one stablished by the signature, although it does not generate any compile error or warning:

```swift
class BaseError: Error { init() { ... } }
class ConcreteError: BaseError { }

func foo() throws BaseError { throw BaseError() }
do { try foo() }
catch { /* error is BaseError */ }

func baz() throws BaseError { throw ConcreteError() }
do { try baz() }
catch { /* error is BaseError */ }
```

### Multiple throwing types in the same do statement

A plain throwing function that throw different errors inside the same `do` statement are type-erased into the `catch` statement, being possible to (as of today), `catch` concrete errors and handle them if needed.

```swift
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

### Unrelated type casting

A function that `throws` a typed error cannot be catched inside a `do` block using casting to other type unrelated to it, generating an error if occurred.

```swift
do {
   try typedThrow() // Throws MyError
} catch let error as NSError { // error: NSError cannot be casted to MyError
   fatalError() 
}
```

### Type inference

1. A function that `throws` an `enum` based `Error` can avoid to explicitly declare the type, and just leave the case, as the type itself is declared in the function declaration signature.  

```swift
enum Foo: Error { case bar, baz }

func fooThrower() throws Foo {
    guard someCondition else {
        throw .bar
    }

    guard someOtherCondition else {
        throw .baz
    }
}
```

2. Assuming it is the only thrown error type in the `do` block, an `enum` based `Error` thrown by a typed throws can have its cases catched, with each case having a separate catch clause. When catching cases the type of the `case` can be omitted, as it is inferred from the throwing function.

```swift
do { try fooThrower() }
catch .bar { ... }
catch .baz { ... }
```

### Protocols

When protocols are involved, we can define naturally typed `throws` functions using global or `associatedtypes` that conform to `Swift.Error`.

```swift
protocol CatProtocol {
    associatedtype CatError: Error // The explicit Error conformance could be ommited if there's a consumer that define the type as a throwing one.
    
    func feedCat() throws CatError -> CatStatus
}
```

The protocol function with typed `throws` are not restricted to `associatedtype`s and any error being valid in any of the cases above is valid too, doesn't matter the origin of it as far as it satisfies the current visibility restrictions regarding parameters in a function:

```swift
private struct CatError: Error {}

public func feedCat() throws CatError -> CatStatus // error: feedCat cannot be declared public because its parameter uses a private type.

// But:
public func feedCat() throws -> CatStatus // Only throws CatError
// Does not generate any compiler warning nor error.
```


> // TODO: Where to put this?

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

### Consistency accross error handling

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

### Library Evolution

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

### Type erasure

Erasing an error type of a function that throws is easy as

```swift
catch {
    // assume the error is inferred as `CatError`
    let typeErasedError: Error = error
}
```

### `rethrow` (generic errors in map, filter, etc)`

While affecting to the `throws` keyword, there should be to have in consideration the `rethrows` clause when a type is present in the method signature, altough there is no impact over `rethrows`, as it inherits from its inner throwing type: 

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

All the rules explained above apply to `rethrows`, and from this proposal we think that they have to receive the same attributes.

### Generics

There's room also for generic errors, where automatically are constrained to `Error`, but the developer can constraint them to more specific errors to generate more fine-grained error throwing thanks to inheritance.

```swift
protocol BaseError: Error {
    var underlyingReason: String { get }
}

protocol ConcreteError: BaseError {}

enum DomainError: ConcreteError {
    case foo
    
    var underlyingReason: String {
        var description = ""
        dump(error, to: &description)
        return description
    }
}

struct OneError: BaseError {
    var underlyingReason: String {
        "foo"
    }
}

func foo<T: BaseError>() throws T { ... }

// The generic gets resolved as `OneError` which implements `BaseError`, 
// function signature changes to `foo() throws OneError`.
do { try foo(OneError()) }
catch { error is `OneError` }

// The generic gets resolved as `DomainError` which implements `ConcreteError` which implements `BaseError`,
// function signature changes to `foo() throws DomainError
do { try foo(DomainError.foo) }
catch { error is `DomainError` }
```

There has to be noted that there's a difference between the rules that apply to generics in Swift nowadays besides what is being proposed because there's a restriction where you cannot declare a generic which type is not used in the function signature, that applies even to the return type of the method if it has not been used in the parameter list before. This rule does not apply in typed `throws`, where you can specify a generic that is being only used as constraint for the throwing type and not in the function signature itself. This generic acts as if it were a common generic and can be used in the function body or even in the return type or in the parameter list as long as the rules that apply to the proposal are satisfied.

[Type inference](#type-inference), a topic that has been already discussed, has its place when generics are involved and, where more power to the type system they provide. This allows us to write code like the following without any compromise.

```swift
enum E: Error { case failure }

func f<T>(_ t: () throws T -> Void) { print(T.self) }

f({ throw E.failure }) // closure gets inferred to be `() throws E -> Void` so this will compile fine 
// Prints: E.Type
```

### Error structure

As explained in [the grammar rules](#the-grammar-rules): Everything that applies to the error type of `Result` also applies to error type of `throws`.

## Source compatibility

Being this change purely additive it would not affect on source compatibility.
Nevertheless, warnings might be produced in different scenarios like the examine below.

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

Which means that by changing to the latter function clients rebuilding will get a warning (saying that error is `Foo`).
So developers may have in consideration the addition of types to a plain throwing functions and deide whether the change will affect negatively in the client's code.

Consider important that the major side-effect regarding source compatibility is the addition of warnings on some specific scenarios like the represented above, which means that the code will remain executing the same way as before and hence don't breaking any current behaviour related with the change.

## Effect on ABI stability

No known effect.

## Effect on API resilience

No known effect.

## Alternatives considered

