# Typed throws

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Jorge Revuelta (@minuscorp)](https://github.com/minuscorp), [Torsten Lehmann](https://github.com/torstenlehmann)
* Review Manager: TBD
* Status: **Proposed**



## Introduction

Swift's error handling model allows functions and closures marked `throws` to note that they can exit by throwing an error. The error values themselve are always type-erased to `any Error`. This approach encourages errors to be handled generically, but makes it impossible to provide more precisely-typed errors without resorting to something like [`Result`](https://developer.apple.com/documentation/swift/result). This proposal introduces the ability to specify that functions and closures only throw errors of a particular concrete type.

Swift-evolution threads:

* [Typed throw functions - Evolution / Discussion - Swift Forums](https://forums.swift.org/t/typed-throw-functions/38860)
* [Status check: typed throws](https://forums.swift.org/t/status-check-typed-throws/66637)


## Motivation

Swift is known for being explicit about semantics and using types to communicate constraints that apply to specific structures and APIs. Some developers are not satisfied with the current state of `throws` as it is not explicit about errors that are thrown. These leads to the following issues with `throws` current behaviour.

### Communicates less error information than `Result` or `Task`

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
func callFutureCat() -> Task<Cat, CatError>
```

with

```swift
func callCatOrThrow() throws -> Cat
```

`throws` communicates less information about why the cat is not about to come to you.

### Inability to interconvert `throws` with `Result` or `Task`

The fact that`throws` carries less information than `Result` or `Task` means that conversions to `throws` loses type information, which can only be recovered by explicit casting:

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

### `Result` is not the go to replacement for `throws` in imperative languages

Using explicit errors with `Result` has major implications for a code base. Because the exception handling mechanism ("goto catch") is not built into the language (like `throws`), you need to do that on your own, mixing the exception handling mechanism with domain logic.

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

### Existential error types incur overhead

Untyped errors have the existential type `any Error`, which incurs some [necessary overhead](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md), in code size, heap allocation overhead, and execution performance, due to the need to support values of unknown type. In constrained environments such as those supported by [Embedded Swift](https://forums.swift.org/t/embedded-swift/67057), existential types may not be permitted due to these overheads, making the existing untyped throws mechanism unusable in those environments.

### Patterns of Swift libraries

There are specific error types in typical Swift libraries like [DecodingError](https://developer.apple.com/documentation/swift/decodingerror), [CryptoKitError](https://developer.apple.com/documentation/cryptokit/cryptokiterror) or [ArchiveError](https://developer.apple.com/documentation/applearchive/archiveerror). But it's not visible without documentation, where these errors can emerge.

On the other hand error type erasure has it's place. If an extension point for an API should be provided, it is often to restrictive to expect specific errors to be thrown. `Decodable`s [init(from:)](https://developer.apple.com/documentation/swift/decodable/2894081-init) may be too restrictive with an explicit error type provided by the API.

Like it's layed out in [ErrorHandlingRationale](https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst) there is valid usage for optionals and throws and we propose even for a typed throws. It comes down to how explicit an API should be and this can vary substantially based on requirements.


## Proposed solution

In general we want to add the possibility to use `throws` with a single, specific error.

```swift
func callCat() throws(CatError) -> Cat {
  if Int.random(in: 0..<24) < 20 {
    throw .sleeps
  }
  // ...
}
```

The function can only throw instances of `CatError`. This provides contextual type information for all throw sites, so we can write `.sleeps` instead of the more verbose `CatError.sleeps` that's needed with untyped throws. Any attempt to throw any other kind of error out of the function will be an error:

```swift
func callCatBadly() throws(CatError) -> Cat {
  throw GenericError(message: "sleeping")  // error: GenericError cannot be converted to CatError
}
```

Maintaining specific error types throughout a function is much easier than when using `Result`, because one can use `try` consistently:

```swift
func stringFromArray(_ array: [String], at index: Int, errorMessage: String) throws(GenericError) -> String {
    guard array.indices.contains(index) else { throw GenericError(message: errorMessage) }
    return array[index]
}

func userResultFromStrings(strings: [String]) throws(GenericError) -> User  {
    let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
    let lastName = try stringFromArray(strings, at: 1, errorMessage: "Missing last name")
    return User(firstName: firstName, lastName: lastName)
}
```

The error handling mechanism is pushed aside and you can see the domain logic more clearly. 

### Concrete error types in catch blocks

With typed throws, a throwing function contains the same information about the error type as `Result`, making it easier to convert between the two:

```swift
func callAndFeedCat1() -> Result<Cat, CatError> {
    do {
        return Result.success(try callCat())
    } catch {
        // would compile now, because error is `CatError`
        return Result.failure(error)
    }
}
```

Note that the implicit `error` variable within the catch block has the concrete type `CatError`; there is no need for the existential `any Error`. 

When a `do` statement can throw errors with different concrete types, or involves any calls to functions using untyped throws, the `catch` block will receive an `any Error` type:

```swift
func callKids() throws(KidError) -> [Kid] { ... }

do {
  try callCat()
  try callKids()
} catch {
  // error has type 'any Error', as it does today
}
```

When one needs to translate errors of one concrete type to another, use a `do...catch` block around each sequence of calls that produce the same kind of error :

```swift
func firstNameResultFromArray(_ array: [String]) throws(FirstNameError) -> String {
    guard array.indices.contains(0) else { throw FirstNameError() }
    return array[0]
}

func userResultFromStrings(strings: [String]) throws(GenericError) -> User  {
    do {
        let firstName = try stringFromArray(strings, at: 0, errorMessage: "Missing first name")
        return User(firstName: firstName, lastName: "")        
    } catch {
        // error is a `FirstNameError`, map it to a `GenericError`.
        throw GenericError(message: "First name is missing")
    }
}
```

### Throwing `any Error` or `Never`

Typed throws generalizes over both untyped throws and non-throwing functions. A function specified with `any Error` as its thrown type:

```swift
func throwsAnything() throws(any Error) { ... }
```

is equivalent to untyped throws:

```swift
func throwsAnything() throws { ... }
```

Similarly, a function specified with `Never` as its thrown type:

```swift
func throwsNothing() throws(Never) { ... }
```

is equivalent to a non-throwing function:

```swift
func throwsNothing() { }
```

There is a more general subtyping rule here that says that you can loosen the thrown type, i.e., converting a non-throwing function to a throwing one, or a function that throws a concrete type to one that throws `any Error`. 

### Interconverting between throwing functions and `Result`

Swift's `Result` type has an [`init(catching:)`](https://github.com/apple/swift-evolution/blob/main/proposals/0235-add-result.md#detailed-design) operation that produces a result when calling a throwing closure, but that result always has the existential type `any Error` as its failure type. With this proposal, we can generalize that operation to produce a more specific result:

```swift
extension Result {
  init(catching body: throws(Failure) -> Success) { ... }
}
```

Now, the expression `Result(catching: callCat)` will produce an instance of type `Result<Cat, CatError>`, relying on type inference to propagate the thrown error type from `callCat` to the `Failure` type. The aforementioned relationship between thrown types specified as `any Error` and `Never` makes this new formulation subsume existing use cases.

## Detailed design

### Syntax adjustments

The [Swift grammar](https://docs.swift.org/swift-book/ReferenceManual/zzSummaryOfTheGrammar.html) is updated wherever there is either `throws` or `rethrows`, to optionally include a thrown type, e.g.,

```
throws-clause -> throws thrown-type(opt)

thrown-type -> '(' type ')'
```

#### Function type

Changing from

```
function-type → attributes(opt) function-type-argument-clause async(opt) throws(opt) -> type
```

to

```
function-type → attributes(opt) function-type-argument-clause async(opt) throws-clause(opt) -> type
```

Examples

```swift
() -> Bool
() throws -> Bool
() throws(CatError) -> Bool
```

#### Closure expression

Changing from

```
closure-signature → capture-list(opt) closure-parameter-clause async(opt) throws(opt) function-result opt in
```

to

```
closure-signature → capture-list(opt) closure-parameter-clause async(opt) throws-clause(opt) function-result opt in
```

Examples

```swift
{ () -> Bool in true }
{ () throws -> Bool in true }
{ () throws(CatError) -> Bool in true }
```


#### Function, initializer, and accessor declarations

Changing from

```
function-signature → parameter-clause async(opt) throws(opt) function-result(opt)
function-signature → parameter-clause async(opt) rethrows(opt) function-result(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause async(opt) throws(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause async(opt) throws(opt)
```

to

```
function-signature → parameter-clause async(opt) throws-clause(opt) function-result(opt)
initializer-declaration → initializer-head generic-parameter-clause(opt) parameter-clause async(opt) throws-clause(opt)
```

Note that the current grammar does not account for throwing accessors, although they should receive the same transformation.

#### Examples

```swift
func callCat() -> Cat
func callCat() throws -> Cat
func callCat() throws(CatError)  -> Cat

init()
init() throws
init() throws(CatError)

var value: Success {
  get throws(Failure) { ... }
}
```

### Throwing within a function that declares a typed error

Any function, closure or function type that is marked as `throws` can declare which type the function throws. That type, which is called the *thrown error type*, must conform to the `Swift.Error` protocol.

Every uncaught error that can be thrown from the body of the function must be convertible to the thrown error type. This applies to both explicit `throw` statements and any errors thrown by other calls (as indicated by a `try`). For example:

```swift
func throwingTypedErrors() throws(CatError) {
  throw CatError.asleep // okay, type matches
  throw .asleep // okay, can infer contextual type from the thrown error type
  throw KidError() // error: KidError is not convertible to CatError
  
  try callCat() // okay
	try callKids() // error: implicitly throws KidError, which is not convertible to CatError
  
  do {
    try callKids() // okay, because this error is caught and suppressed below
  } catch {
    // eat the error
  }
}
```

Because a value of any `Error`-conforming type implicitly converts to `any Error`, this implies that an function declared with untyped `throws` can throw anything:

```swift
func untypedThrows() throws {
  throw CatError.asleep // okay, CatError converts to any Error
  throw KidError() // okay, KidError converts to any Error
  try callCat() // okay, thrown CatError converts to any Error
	try callKids() // okay, thrown KidError converts to any Error
}
```

Therefore, these rules subsume those of untyped throws, and no existing code will change behavior.

### Catching typed thrown errors 

A `do...catch` block is used to catch and process thrown errors. With only untyped errors, the type of the error thrown from inside the `do` block is always `any Error`. In the presence of typed throws, the type of the error thrown from inside the `do` block depends on the specific throwing sites. 

When all throwing sites within a `do` block produce the same error type (ignoring any that throw `Never`), that error type is used as the type of the thrown error. For example:

```swift
do {
  try callCat() // throws CatError
  if something {
    throw CatError.asleep // throws CatError
  }
} catch {
  // implicit 'error' value has type CatError
  if error == .asleep { 
    openFoodCan()
  }
}
```

This also implies that one can use the thrown type context to perform type-specific checks in the catch clauses, e.g.,

```swift
do {
  try callCat() // throws CatError
  if something {
    throw CatError.asleep // throws CatError
  }
} catch .asleep {
  openFoodCan()
} // note: CatError can be thrown out of this do...catch block when the cat isn't asleep**Rationale**: 
```

> **Rationale**: By inferring a concrete result type for the thrown error type, we can entirely avoid having to reason about existential error types within `catch` blocks, leading to a simpler syntax. Additionally, it preserves the notion that a `do...catch` block that has a `catch` site accepting anything (i.e., one with no conditions) can exhaustively suppress all errors. 

When throw sites within the `do` block throw different (non-`Never`) error types, the resulting error type is `any Error`. For example:

```swift
do {
  try callCat() // throws CatError
  try callKids() // throw KidError
} catch {
  // implicit 'error' variable has type 'any Error'
}
```

In essence, when there are multiple possible thrown error types, we immediately resolve to the untyped equivalent of `any Error`. Again, this rule for typed throws subsumes the existing rule for untyped throws, in which every throw site produces an error of type `any Error`.

> **Rationale**: While it would be possible to compute a more precise "union" type of different error types, doing so is potentially an expensive operation at compile time and run time, as well as being harder for the programmer to reason about. If in the future it becomes important to tighten up the error types, that could be done in a mostly source-compatible manner.

### `rethrows`

The adjustments to `rethrows` differ depending on how many different errors are thrown by the typed `throws` of the inner functions.

With **no** error being thrown by the inner functions `rethrows` also does not throw an error.

If there is **one** error of type `E` `rethrows` will also throw `E`.

```swift
func foo<E>(closure: () throws(E) -> Void) rethrows // throws E
```

In the example above there's no need to constraint `E: Error`, as any other kind of object that does not conform to `Error` will throw a compilation error, but it is handy to match the inner `Error` with the outer one. So the set of functions in the Standard Library (`map`, `flatMap`, `compactMap` etc.) that support `rethrows`, can be advanced to their error typed versions just by modifying the signature like

```swift
// current
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]

// updated to
func map<T, E>(_ transform: (Element) throws(E) -> T) rethrows -> [T]
```

If there are only **multiple errors of the same type** `rethrows` throws an error of the same type.

```swift
func foo<E>(f: () throws(E) -> Void, g: () throws(E) -> Void) rethrows // throws E
```

If there are **multiple differing errors** `rethrows` just throws `Error`.

```swift
func foo<E1, E2>(f: () throws(E1) -> Void, g: () throws(E2) -> Void) rethrows // throws Error
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

func foo<E1, E2>(f: () throws(E1) -> Void, g: () throws(E2) -> Void) rethrows // throws ErrorUnion2<E, E2> -> Void
```

But for `rethrows` to work in this way, these `enum`s need to be part of the standard library. A downside to mention is, that an `ErrorUnion2` would not be apple to auto merge its cases into one, if the cases are of the same error type, where with sum types `A | A === A`.

### Usage in Protocols

We can define typed `throws` functions in protocols with specific error types that are visible to the caller

```swift
private struct PrivateCatError: Error {}
public struct PublicCatError: Error {}

protocol CatFeeder {
    public func throwPrivateCatErrorOnly() throws -> CatStatus // compiles
    public func throwPrivateCatErrorExplicitly() throws (PrivateCatError) -> CatStatus // won't compile 
    public func throwPublicCatErrorExplicitly() throws (PublicCatError) -> CatStatus // compiles
}
```

Or we can use `associatedtypes` that (implicitly) conform to `Swift.Error`.

```swift
protocol CatFeeder {
    associatedtype CatError: Error // The explicit Error conformance can be omited if there's a consumer that defines the type as a thrown one.
    
    func feedCat() throws(CatError) -> CatStatus
}
```

### Usage with generics

Typed `throws` can be used in combination with generic functions by making the error type generic.

```swift
func foo<E>(e: E) throws(E)
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
let f2: () throws(SubError) -> Void = f1
```

It's also allowed to assign a subtype of a thrown error, though the subtype information is erased and the error of f2 will be casted up.

```swift
let f3: () throws(BaseError) -> Void = f2
```

Erasing the specific error type is possible

```swift
let f4: () throws -> Void = f3
```

In general (assuming function parameters and return type are compatible):

- `() -> Void` is subtype of `() throws(B) -> Void`
- `() throws(B) -> Void` is subtype of `() throws -> Void`
- `() throws(B) -> Void` is subtype of `() throws(A) -> Void` if `B` is subtype of `A`

`#openquestion` For the semantic analysis it was suggested that every function is interpreted as a throwing function leading to this equivalences

```swift
() -> Void === () throws(Never) -> Void
() throws -> Void === () throws(Error) -> Void
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
    func f() throws(ColoredError)
}

protocol ThrowingBlueError: ThrowingColoredError {
    // Refinement
    func f() throws(BlueError)
}

protocol ThrowingDeepBlueErrorError: ThrowingBlueError {
    // Refinement
    func f() throws(DeepBlueError)
}
```

### Type inference for `enum`s

A function that `throws` an `enum` based `Error` can avoid to explicitly declare the type, and just leave the case, as the type itself is declared in the function declaration signature.  

```swift
enum Foo: Error { case bar, baz }

func fooThrower() throws(Foo) {
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
func getCatOrThrow() throws(CatError) -> Cat
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
init(catching body: () throws(Failure) -> Success)
```

and `Result`'s [get()](https://developer.apple.com/documentation/swift/result/3139397-get) from

```swift
func get() throws -> Success
```

to

```swift
func get() throws(Failure) -> Success
```

### Library Evolution

There are many concerns about library evolution and compatibility.

#### Non `@frozen enum`s

Our approach is quite similar to what happens with switch cases:

```swift
enum NonFrozenEnum: Error { case cold, warm, hot }

func wheathersLike() throws(NonFrozenEnum) -> Weather

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
    func load throws(DataLoaderError) -> Data
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

func theErrorMaker() throws(SomeErrors)

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
var throwers = [{ throw Foo() }] // Inferred as `Array<() throws -> ()>`, or `Array<() throws(Foo) -> ()>`?
throwers.append({ throw Bar() }) // Compiles today, error if we infer `throws(Foo)`
```

The combination of type inference and `throw` can lead to trouble, because with more specific error types supported for throwing statements, the inferred type needs to change to the more specific type. At least this would be the intuitive behaviour.

Another example would be updating Result's [init(catching:)](https://developer.apple.com/documentation/swift/result/3139399-init) and [get()](https://developer.apple.com/documentation/swift/result/3139397-get).

**In general: All locations where an error will be inferred or updated to a more specific error can cause trouble with source compatibility.**

## Effect on ABI stability

[swift/Mangling.rst at master · apple/swift](https://github.com/apple/swift/blob/master/docs/ABI/Mangling.rst)

```
function-signature ::= params-type params-type async? throws-clause?
```

`#openquestion` Any insights are appreciated.

## Effect on API resilience

`#openquestion` Any insights are appreciated.

## Alternatives considered

See [Motivation](#motivation)
