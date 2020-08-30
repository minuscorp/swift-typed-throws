# Typed throws

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Jorge Revuelta (@minuscorp)](https://github.com/minuscorp), [Torsten Lehmann](https://github.com/torstenlehmann)
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

As a user of `callFamily() throws` it gets a lot harder to understand, which errors can be thrown. Even if reading the functions implementation would be possible (which sometimes is not), then you are usally forced to read the whole implementation, collecting all uses of `try` and investigating the whole error flow dependencies of the sub program. Which almost nobody does, and the people that try often produce mistakes, because complexity quickly outgrows.

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

API documentation could and usually become outdated, because it's not checked by the compiler. Furthermore the `- throws:` documentation does not provide linking to errors ([Apple Developer Markup Formatting Reference](https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_markup_formatting_ref/index.html#//apple_ref/doc/uid/TP40016497-CH2-SW1)) making it even harder to find the error types in questions.

Assume some scarce documentation (more thorough documentation is even more likely to get outdated).

```swift
/// - throws: CatError
func callCatOrThrow() throws -> Cat
```

Let's update the method to load this cat from the network:

```swift
/// - throws: CatError
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

Now we have outdated catch clauses

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

### Patterns of Swift libraries

There are specific error types in typical Swift libraries like [DecodingError](https://developer.apple.com/documentation/swift/decodingerror), [CryptoKitError](https://developer.apple.com/documentation/cryptokit/cryptokiterror) or [ArchiveError](https://developer.apple.com/documentation/applearchive/archiveerror). But it's not visible without documentation, where these errors can emerge.

On the other hand error type erasure has it's place. If an extension point for an API should be provided, it is often to restrictive to expect specific errors to be thrown. `Decodable`s [init(from:)](https://developer.apple.com/documentation/swift/decodable/2894081-init) may be to restrictive with an explicit error type provided by the API.

Like it's layed out in [ErrorHandlingRationale](https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst) there is valid usage for optionals and throws and we propose even for a typed throws. It comes down to how explicit an API should be and this can vary substantially based on requirements.


## Proposed solution

In general we want to add the possibility to use `throws` with a single, specific error.

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
    } catch let error as CatError {
        // would compile now, because error is `CatError`
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

and you make sure that you are aware of a possible `CatError` by explicitly catching it

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

You can now update your `catch` clauses to make sure that you are aware of the new error cases and that you handle them properly.

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

Be aware that in all 3 approaches we are omitting the issue of simplifying error type conversions, which can be a topic for another proposal. But here's how it would look like for Approach 1 and 3 without further language constructs.

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
    } catch let error as FirstNameError {
        // Mapping from `FirstNameError` to a `GenericError`
        throw GenericError(message: "First name is missing")
    }
}

```

An example with multiple errors can be found here:

https://forums.swift.org/t/typed-throw-functions/38860/122.


## Detailed design

### Syntax adjustments

We are referring to [Summary of the Grammar — The Swift Programming Language (Swift 5.3)](https://docs.swift.org/swift-book/ReferenceManual/zzSummaryOfTheGrammar.html).

Adding

```
throws-clause -> throws type-identifier(opt)
```

to the grammar.

#### Changes to grammar

Changing from:

```
function-type → attributes(opt) function-type-argument-clause throws(opt) -> type
closure-signature → capture-list(opt) closure-parameter-clause throws(opt) function-result opt in
function-signature → parameter-clause throws(opt) function-result(opt)
protocol-initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause throws(opt) generic-where-clause(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause throws(opt)
```

to:

```
function-type → attributes(opt) function-type-argument-clause throws-clause(opt) -> type
closure-signature → capture-list(opt) closure-parameter-clause throws-clause(opt) function-result opt in
function-signature → parameter-clause throws-clause(opt) function-result(opt)
protocol-initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause throws-clause(opt) generic-where-clause(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause throws-clause(opt)
```

### Rules for `throws` and `catch`

#### `throws`

1. Any function/method, (protocol) init method, closure or function type that is marked as `throws` can declare which type the function/method throws.
2. At most one type of specific error can be used with a `throws`.
3. The error **must** conform to `Swift.Error` by (transitive) conformation.
4. An error thrown from the function's body has to be compatible with the thrown error of the function's signature.

#### `catch`

1. Throwing inside the `do` block using `throws` or a function that `throws` is handled the same regarding errors.
2. A general `catch` clause always infers the `error` as `Swift.Error`
3. `#openquestion` In general needless `catch` clauses are marked with warnings (prefering more specific ones to keep if there is a conflict between clauses). But it should be discussed for which scenarios we can apply these, because it's not easy do decide this for non trivial `catch` clauses or error type hierarchies.

`#openquestion` Alternative to consider:

> If all statements in the `do` block throw specific errors and there is a `catch` clause that does not match one of this errors, then a compiler error is generated.

### Error scenarios considered

Assuming the functions and errors

```swift
func callCat() throws CatError -> Cat
func callKids() throws KidsError -> Kids

struct CatError {
    reason: String
}

struct KidsError {
    reason: String
}
```

#### Scenario 1: Specific thrown error, general catch clause

```swift
do {
    let cat = try callCat()
} catch {
    // error is inferred as `Swift.Error` to keep source compatibility
}
```

#### Scenario 2: Specific thrown error, specific catch clause

```swift
do {
    let cat = try callCat()
} catch let error as CatError { // ensure `CatError` even if the API changes in the future
    // error is `CatError`
    // so this would compile
    let reason = error.reason
}
```

No general catch clause needed. If there is one, compiler will show a warning (comparable to `default` in `switch`).

#### Scenario 3: Specific thrown error, multiple catch clauses for one enum

```swift
// Assuming an enum `CatError`
enum CatError: Error {
    case sleeps, sitsOnATree
}

do {
    let cat = try callCat()
} catch .sleeps { // Type inference makes the catch clause more dense
    // handle error
} catch .sitsOnATree {
    // handle error
}
```

#### Scenario 4: Multiple same specific thrown errors, specific catch clause

```swift
do {
    let cat = try callCat()
    throw CatError
} catch let error as CatError {
    // error is `CatError`
}
// this is exhaustive
```

#### Scenario 5: Multiple differing specific thrown errors, general catch clause

```swift
do {
    let kids = try callKids()
    let cat = try callCat()
} catch {
    // `error` is just the type erased `Swift.Error`
    // because we can't auto generate KidsError | CatError
    // and we want to keep source compatibility
}
```

#### Scenario 6: Multiple differing specific thrown errors, multiple specific catch clauses

```swift
do {
    let kids = try callKids()
    let cat = try callCat()
} catch let error as CatError {
    // `error` is `CatError`
} catch let error as KidsError {
    // `error` is `KidsError `
}
```

#### Scenario 7: Multiple specific thrown errors, specific and general catch clauses

```swift
do {
    let kids = try callKids()
    let cat = try callCat()
} catch let error as CatError {
    // `error` is `CatError`
} catch {
    // `error` is `Swift.Error `
}
```

#### Scenario 8: Unspecific thrown error

- Current behaviour of Swift applies

### `rethrows`

The adjustments to `rethrows` differ depending on how many different errors are thrown by the typed `throws` of the inner functions.

With **no** error being thrown by the inner functions `rethrows` also does not throw an error.

If there is **one** error of type `E` `rethrows` will also throw `E`.

```swift
func foo<E>(closure: () throws E -> Void) rethrows // throws E
```

In the example above there's no need to constraint `E: Error`, as any other kind of object that does not conform to `Error` will throw a compilation error, but it is handy to match the inner `Error` with the outer one.

This behavior mimics the current one dictated by:

```swift
func bar(closure1: () throws -> Void, closure2: () throws -> Void) rethrows
```

That stablishes the following rules:

1. If no closure throws, bar does not throw,
2. If all throwing closures are typed-throw with the same error type `E`, **bar** throws `E`,
3. If throwing closures throw different error types, or some closures throw untyped errors, **bar** throws an untyped error (`Error`).

Those rules are extended to the usage of generics as follows:

If there are only **multiple errors of the same type** `rethrows` throws an error of the same type.

```swift
func foo<E>(f: () throws E -> Void, g: () throws E -> Void) rethrows // throws E
```

If there are **multiple differing errors** `rethrows` just throws `Error`.

```swift
func foo<E1, E2>(f: () throws E1 -> Void, g: () throws E2 -> Void) rethrows // throws Error
```

Because information loss will happen by falling back to `Error` this solution is far from ideal, because keeping type information on errors is the whole point of the proposal.

(Theoretical) Alternatives for rethrowing **multiple differing errors**:

- infer the closest common base type (which seems to be hard, because as to commenters in the forum type relation information seems to be missing in the runtime, however information loss on thrown error types will happen too)
- **Not for discussion of this proposal:** sum types like `A | B` which were discussed and rejected in the past (see [swift-evolution/xxxx-union-type.md](https://github.com/frogcjn/swift-evolution/blob/master/proposals/xxxx-union-type.md))
- use some replica sum type `enum` like

```swift
enum ErrorUnion2<E1: Error, E2: Error>: Error {
    case first(E1)
    case second(E2)
}

func foo<E1, E2>(f: () throws E1 -> Void, g: () throws E2 -> Void) rethrows // throws ErrorUnion2<E, E2> -> Void
```

But for `rethrows` to work in this way, these `enum`s need to be part of the standard library. A downside to mention is, that an `ErrorUnion2` would not be apple to auto merge its cases into one, if the cases are of the same error type, where with sum types `A | A === A`.

### Usage in Protocols

We can define typed `throws` functions in protocols with specific error types that are visible to the caller

```swift
private struct PrivateCatError: Error {}
public struct PublicCatError: Error {}

protocol CatFeeder {
    public func throwPrivateCatErrorOnly() throws -> CatStatus // compiles
    public func throwPrivateCatErrorExplicitly() throws PrivateCatError -> CatStatus // won't compile 
    public func throwPublicCatErrorExplicitly() throws PublicCatError -> CatStatus // compiles
}
```

Or we can use `associatedtypes` that (implicitly) conform to `Swift.Error`.

```swift
protocol CatFeeder {
    associatedtype CatError: Error // The explicit Error conformance can be omited if there's a consumer that defines the type as a thrown one.
    
    func feedCat() throws CatError -> CatStatus
}
```

### Usage with generics

Typed `throws` can be used in combination with generic functions by making the error type generic.

```swift
func foo<E>(e: E) throws E
```

`E` would be constrained to `Error`, because it is used in `throws`.

### Subtyping

#### Between functions

Having related errors and a non-throwing function

```swift
class BaseError: Error {}
class SubError: BaseError {}

let f1: () -> Void
```

Converting a non-throwing function to a throwing one is allowed

```swift
let f2: () throws SubError -> Void = f1
```

It's also allowed to assign a subtype of a thrown error, though the subtype information is erased and the error of f2 will be casted up.

```swift
let f3: () throws BaseError -> Void = f2
```

Erasing the specific error type is possible

```swift
let f4: () throws -> Void = f3
```

In general (assuming function parameters and return type are compatible):

- `() -> Void` is subtype of `() throws B -> Void`
- `() throws B -> Void` is subtype of `() throws -> Void`
- `() throws B -> Void` is subtype of `() throws A -> Void` if `B` is subtype of `A`

`#openquestion` For the semantic analysis it was suggested that every function is interpreted as a throwing function leading to this equivalences

```swift
() -> Void === () throws Never -> Void
() throws -> Void === () throws Error -> Void
```

But it should be discussed if these equivalences should become part of the syntax.

#### Catching errors that are subtypes

Following the current behaviour of `catch` clauses the first clause that matches is chosen.

```swift
class BaseError: Error {}
class SpecificError: BaseError {}

func throwBase() throws {
    throw SpecificError()
}

do {
    try throwBase()
} catch let error as SpecificError {
    print("specific") // uses this clause
} catch let error as BaseError {
    print("base")
}

do {
    try throwBase()
} catch let error as BaseError {
    print("base") // uses this clause
} catch let error as SpecificError {
    print("specific")
}
```

#### Protocol refinements

Protocols should have the possibility to conform and refine other protocols containing throwing functions based on the subtype relationship of their functions. This way it would be possible to throw a more specialised error or don't throw an error at all.

Examples from https://forums.swift.org/t/typed-throw-functions/38860/223

```swift
protocol Throwing {
    func f() throws
}

protocol NotThrowing: Throwing {
    // A non-throwing refinement of the method
    // declared by the parent protocol.
    func f()
}
```

```swift
protocol ColoredError: Error { }
class BlueError: ColoredError { }
class DeepBlueError: BlueError { }

protocol ThrowingColoredError: Throwing {
    // Refinement
    func f() throws ColoredError
}

protocol ThrowingBlueError: ThrowingColoredError {
    // Refinement
    func f() throws BlueError
}

protocol ThrowingDeepBlueErrorError: ThrowingBlueError {
    // Refinement
    func f() throws DeepBlueError
}
```

### Type inference for `enum`s

A function that `throws` an `enum` based `Error` can avoid to explicitly declare the type, and just leave the case, as the type itself is declared in the function declaration signature.  

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

Assuming it is the only thrown error type in the `do` block, an `enum` based `Error` can have its cases catched, with each case having a separate `catch` clause. When catching cases the type of the `case` can be omitted, as it is inferred from the throwing function.

```swift
do { try fooThrower() }
catch .bar { ... }
catch .baz { ... }
```

### Converting between `throws` and `Result`

Having a typed `throws` it would be quite convenient to not being forced to explicitly convert between `throws` and `Result`. Semantically `throws` could be just another syntax for `Result`, which would make both of them more composable.

```swift
func getCatOrThrow() throws CatError -> Cat
func getCatResult() -> Result<Cat, CatError>
```

So it would be nice if we could

```swift
do {
    let cat1: Cat = try getCatOrThrow() // works as always
    let cat2: Cat = try getCatResult() // `try` will unwrap the `Result` by calling `Result.get()`
    let catResult1: Result<Cat, CatError> = getCatResult()
    let catResult2: Result<Cat, CatError> = getCatOrThrow() `throws` is interpreted as the corresponding `Result`
    let cat3: Cat = getCatOrThrow().get() // same as `try getCatOrThrow()`
} catch let error as CatError {
    ...
}
```

But from what we know, this was already discussed before and was rejected in favour of a performant `throws` implementation.

But at least we recommend updating `Result`'s [init(catching:)](https://developer.apple.com/documentation/swift/result/3139399-init) from

```swift
init(catching body: () throws -> Success)
```

to

```swift
init(catching body: () throws Failure -> Success)
```

and `Result`'s [get()](https://developer.apple.com/documentation/swift/result/3139397-get) from

```swift
func get() throws -> Success
```

to

```swift
func get() throws Failure -> Success
```

### Library Evolution

There are many concerns about library evolution and compatibility.

#### Non `@frozen enum`s

Our approach is quite similar to what happens with switch cases:

```swift
enum NonFrozenEnum: Error { case cold, warm, hot }

func wheathersLike() throws NonFrozenEnum -> Weather

try { wheathersLike() } 
catch .cold { ... }
catch .warm { ... }
catch .hot { ... } // warning: all cases were catched but NonFrozenEnum might have additional unknown values.
// So if the warning is resolved:
catch let error as NonFrozenEnum { ... }
```

So it maintains backwards compatibility emiting a warning instead of an error. An error could be generated if this proposal doesn't need to keep backwards compatible with previous Swift versions.

#### API Developer recommendations

Assuming a current API

```swift
struct DataLoaderError {}

protocol DataLoader {
    func load throws DataLoaderError -> Data
}
```

Here are some things to consider when developing an API with typed errors:

- If you need to throw a new specific error, because you think your API user needs to know, that this specific error (e.g. `FileSystemError`) did happen, then it's a breaking change, because your API user may want to react to it in another way now.

Changing 

```swift
struct DataLoaderError {
    let message: String
}
```

to

```swift
enum DataLoaderError {
    case loadError(LoadError)
    case fileSystemError(FileSystemError)
}
```

- If you don't need to throw it, because you think your API user does not need to know this error, then you map it to the error that represents the `FileSystemError` (most of the time in a more abstract sense). In this example you would throw a `DataLoaderError` with another message.

- If you think you don't know what will happen in the future and breaking changes should be avoided as much as possible, then just throw `Swift.Error`. But keep in mind that you are less explicit about what can happen and also take the possibility for the API user to rely on the existence of the error (by using the compiler) leading to issues mentioned in [Motivation](#motivation).

- If you provide an extension point to your API like a protocol (e.g. a `DataLoader` like above) that can be used to customize the behaviour of your API, then try to omit forcing specific errors on the API user. Most of the time you as an extension point provider just want to know that something went wrong. If you need multiple cases of errors then keep the amount as small as possible and eventually do compatibility converting on the API developer side outside of the extension point implementation. "Only ask for what you need" applies here.

### Autocompletion

Because we can't infer types in the general `catch` clause without breaking source compatibility, we are suggesting to use autocompletion while adding missing `catch` clauses. Only `catch` clauses that are missing from the current `do` statement are suggested (including the general `catch` clause).

### Optional Enhancement: Reducing `catch` clause verbosity

`#openquestion`

Because we now need to explicitly catch specific errors in `catch` clauses a lot, a shorter form is being suggested.
Having to write `catch let error as FooError` seems a bit inconsistent with the rest of how `catch` works, as `error` is inferred to be a constant of type `Error` in the general `catch` clause without mentioning `let error`.


```swift
do { ... }
// Here we have to explicitly declare `error` as a constant.
catch let error as FooError { ... }
// Whereas here you have `error` for free.
catch { ... }
```

This inconsistency can induce confusion when writing down different specific and general `catch` clauses, having to declare `error` on your own in one case and omitting it in the other.

For this the grammar would need an update in [catch-pattern](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#grammar_catch-pattern) in the following way:


```
catch-pattern -> type-identifier
```

Example:

```swift
do {
    ...
} catch DogError {
    // `error` is `DogError`
}
```

This comes in handy with `class` and `struct` error types, but most of all with `enum` types.

```swift
enum SomeErrors: Error {
    case foo, bar, baz
}

func theErrorMaker() throws SomeErrors

do {
    try theErrorMaker()
}
catch .foo { ... }
catch .bar { ... }
catch .baz { ... }
```

This behaviour and syntax in general resembles a lot how `switch` cases work because:

1. The type can be ignored, as it is inferred.
2. They must be exhaustive.

In scenarios where different types are involved, each one has the same treatment from the grammar side:

```swift

do {
    try throwsClass() // throws MyClass
    try throwsStruct() // throws MyStruct
    try throwsEnum() // throws MyEnum { case one, two }
} 
catch MyClass { ... }
catch MyStruct { ... }
catch .one { ... }
catch .two { ... } 
```

And where multiple enums are being caught, it would be only needed to specify the type of those cases that were repeated in every enum.

```swift
enum One: Error { case one, two, three }
enum Two: Error { case two, three, four }


do { ... }
catch .one { ... }
catch One.two { ... } // Disambiguate the type.
catch One.three { ... }
catch Two.two { ... }
catch Two.three { ... }
catch .four { ... }
```

These scenarios are uncommon but possible, also there's always room to `catch One` and handle each case in a switch statement.


This change in the expression is merely additive and has no impact on the current source.

## Source compatibility

We decided to keep the inference behaviour of the general `catch` clause (`error: Error`) to keep source compatibility. But if breaking source compatibility is an option, we could change this

```swift
do {
    let cat = try callCat() // throws `CatError`
    throw CatError
} catch let error as CatError {
    // error is `CatError`
}
// this is exhaustive
```

to this

```swift
do {
    let cat = try callCat() // throws `CatError`
    throw CatError
} catch {
    // error is inferred as `CatError`
}
// this is exhaustive
```

There is a scenario, that potentially breaks source compatibility (original post: https://forums.swift.org/t/typed-throw-functions/38860/178):

```swift
struct Foo: Error { ... }
struct Bar: Error { ... }
var throwers = [{ throw Foo() }] // Inferred as `Array<() throws -> ()>`, or `Array<() throws Foo -> ()>`?
throwers.append({ throw Bar() }) // Compiles today, error if we infer `throws Foo`
```

The combination of type inference and `throw` can lead to trouble, because with more specific error types supported for throwing statements, the inferred type needs to change to the more specific type. At least this would be the intuitive behaviour.

Another example would be updating Result's [init(catching:)](https://developer.apple.com/documentation/swift/result/3139399-init) and [get()](https://developer.apple.com/documentation/swift/result/3139397-get).

**In general: All locations where an error will be inferred or updated to a more specific error can cause trouble with source compatibility.**

## Effect on ABI stability

[swift/Mangling.rst at master · apple/swift](https://github.com/apple/swift/blob/master/docs/ABI/Mangling.rst)

```
function-signature ::= params-type params-type throws? throws-type?
```

`#openquestion` Any insights are appreciated.

## Effect on API resilience

`#openquestion` Any insights are appreciated.

## Alternatives considered

See [Motivation](#motivation)
