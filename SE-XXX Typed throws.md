# Typed throws

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Jorge Revuelta (@minuscorp)](https://github.com/minuscorp), [Torsten Lehmann](https://github.com/torstenlehmann), [Doug Gregor](https://github.com/DougGregor)
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
struct GenericError: Error {
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

### Throwing and catching with typed throws

#### Throwing within a function that declares a typed error

Any function, closure or function type that is marked as `throws` can declare which type the function throws. That type, which is called the *thrown error type*, must conform to the `Error` protocol.

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

#### Catching typed thrown errors 

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

In essence, when there are multiple possible thrown error types, we immediately resolve to the untyped equivalent of `any Error`. We will refer to this notion as a type function `errorUnion(E1, E2, ..., EN)`, which takes `N` different error types (e.g., for throwing sites within a `do` block) and produces the union error type of those types. Our definition and use of `errorUnion`  for typed throws subsumes the existing rule for untyped throws, in which every throw site produces an error of type `any Error`.

> **Rationale**: While it would be possible to compute a more precise "union" type of different error types, doing so is potentially an expensive operation at compile time and run time, as well as being harder for the programmer to reason about. If in the future it becomes important to tighten up the error types, that could be done in a mostly source-compatible manner.

The semantics specified here are not fully source compatible with existing Swift code. A `do...catch` block that contains `throw` statements of a single concrete type (and no other throwing sites) might depend on the error being caught as `any Error`. Here is a contrived example:

```swift
do {
  throw CatError.asleep
} catch {
  var e = error   // currently has type any Error, will have type CatError
  e = KidsError() // currently well-formed, will become an error
}
```

> **Swift 6**: To prevent this source compatibility issue, we can refine the rule slightly for Swift 5 code bases to specify that the caught error type must be `any Error` if there are no `try` expressions in the `do` statement. That way, one can only get a caught error type more specific than `any Error` by calling a function that is already making use of typed throws.

#### Typed `rethrows`

A function marked `rethrows` throws only when one of its closure parameters throws. The thrown error type for a particular call to the `rethrows` function depends on the actual arguments to the call, and how typed throws are expressed within the function signature.

For example, consider a function `foo` that takes a closure throwing a type `E` (which maybe be `any Error` or `Never` for untyped throws and non-throwing functions). It will only throw when its argument throws, and then only a value of the thrown type of that argument, `E`:

```swift
func foo<E: Error>(closure: () throws(E) -> Void) rethrows(E)
```

One could have multiple arguments of function type that that all throw the same error type, e.g.,

```swift
func foo<E: Error>(f: () throws(E) -> Void, g: () throws(E) -> Void) rethrows(E)
```

Multiple arguments of function type could throw different error types. The implementation would be responsible either for translating those errors to a specific concrete error type:

```swift
func translateErrors<E1: Error, E2: Error>(
  f: () throws(E1) -> Void, 
  g: () throws(E2) -> Void
) rethrows(GenericError) {
  do {
    try f()
  } catch {
    throw GenericError(message: "E1: \(error)")
  }
  
  do {
    try g()
  } catch {
    throw GenericError(message: "E2: \(error)")
  }
}
```

If called with non-throwing closure arguments, `translateErrors` will not throw. If either of the closure arguments can throw, then `translateErrors` can throw---but will always throw a `GenericError`. 

Alternately, one can specify a suitably-generic thrown error type when rethrowing:

```swift
func untypedOrNonthrowing<E1: Error, E2: Error>(
  f: () throws(E1) -> Void, 
  g: () throws(E2) -> Void
) rethrows(any Error) {
  try f()
  try g()
}
```

If called with non-throwing closure arguments, `untypedOrNonthrowing` will not throw. If either of the closure arguments can throw, then `untypedOrNonthrowing` can throw `any Error`. 

A function can be `rethrows` without specifying any thrown error type. Prior to typed throws, this always meant that it would throw `any Error` if any of the closure arguments throws. For typed throws, we define `rethrows` to be equivalent to `rethrows(errorUnion(E1, E2, ..., EN))`, where each `Ei` is the thrown error type for one of the closure parameters, using the same ideas from `do...catch`statements. This definition is ideal for operations that rethrow only what their closure arguments throw, such as the `map` operation on collections:

```swift
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) rethrows -> [T]
// equivalent to
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) rethrows(E) -> [T]
```

If there are multiple closure parameters that have different thrown error types, the rethrown type will be `any Error`.

```swift
func manyErrors<E1: Error, E2: Error>(
  f: () throws(E1) -> Void, 
  g: () throws(E2) -> Void
) rethrows { ... } // equivalent to rethrows(any Error)
```

>  **Note**: There is a small chance of this rule causing confusion, because while `throws` is equivalent to `throws(any Error)`, `rethrows` will infer the error type differently. It is somewhat contrived, but one could imagine an API that begins like this:
>
> ```swift
> func takeError<E: Error>(f: () throws(E) -> Void) rethrows { // equivalent to rethrows(E)
>   try f()
> }
> ```
>
> possibly evolving to try to translate the error:
>
> ```swift
> func takeError<E: Error>(f: () throws(E) -> Void) rethrows { // equivalent to rethrows(E)
>   do {
> 	  try f()
> 	} catch {
>     throw GenericError(message: "\(error)") // error: GenericError is not an E
>   }
> }
> ```
>
> However, this seems unlikely to cause problems in practice, and the only real way to avoid it would be to prohibit the use of `rethrows` (with no specified thrown type) when any of the closure parameters use typed throws.

The vast majority of `rethrows` functions will directly throw whatever is thrown from their closure parameters, without generating their own errors or performing any translation. However, the simplest formulation of a rethrowing function such as `map`:

```swift
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
```

says that `map` can throw `any Error` whenever its `transform` argument throws an exception, i.e., it will lose typed throws information. A more useful formulation is what is proposed above, i.e.,

```swift
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) rethrows -> [T]
```

which states that `map` will only throw errors of the same type as `transform` does. As a special case, when `rethrows` is not provided with a typed error, we propose to treat any untyped `throws` on a closure parameter as if it were written `throws(Ei)`, where `Ei` is a synthesized generic parameter for that closure parameter `i` that is required to conform to the `Error` protocol. This way, the simplest form of `rethrows` will maintain precise type information. If the body of the function tries to throw anything else (i.e., by doing error translation), it will produce an error.

> **Swift 6**: This rule is convenient, but will break existing code that uses `rethrows` and performs any kind of error translation. Therefore, we delay this specific change in the semantics of untyped `rethrows` with untyped `throws` closure parameters until Swift 6.

#### Opaque thrown error types

The thrown error type of a function can be specified with an [opaque result type](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md). For example:

```swift
func doSomething() throws(some Error) { ... }
```

The opaque thrown error type is like a result type, so the concrete type of the error is chosen by the `doSomething` function itself, and could change from one version to the next. The caller only knows that the error type conforms to the `Error` protocol; the concrete type won't be knowable until runtime.

Due to the contravariance of parameters, an opaque thrown error type that occurs within a function parameter will be an [opaque parameter](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md). This means that the closure argument itself will choose the type, so

```swift
func map<T>(_ transform: (Element) throws(some Error) -> T) rethrows -> [T]
```

is equivalent to

```swift
func map<T, E: Error>(_ transform: (Element) throws(E) -> T) rethrows -> [T]
```

#### Throwing in asynchronous `for..in` loops

The asynchronous `for..in` loop uses an underspecified notion of ["rethrowing" protocol conformances](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md#rethrows) to make it possible for iteration over an asynchronous sequence to be throwing when the sequence's `next()` operation may throw, and non-throwing otherwise. With typed throws, the `AsyncSequence` protocol will gain an associated type `Failure` that provides the thrown error type for the iterator's `next()` operation (see details in the section *Standard library adoption*).

When a `for..in` loop iterators over an `AsyncSequence`, the iteration can throw the `Failure` type of the `AsyncSequence`. When the `Failure` type is `Never`, the loop cannot throw and does not require `try`. This provides a mechanism for handling throwing and non-throwing asynchronous `for..in` loops consistently.

### Subtyping rules

A function type that throws an error of type `A` is a subtype of a function type that differs only in that it throws an error of type `B` when `A` is a subtype of `B`.  As previously noted, a `throws` function that does not specify the thrown error type will have a thrown type of `any Error`, and a non-throwing function has a thrown error type of `Never`. For subtyping purposes, `Never` is assumed to be a subtype of all error types.

The subtyping rule manifests in a number of places, including function conversions, protocol conformance checking and refinements, and override checking, all of which are described below.

#### Function conversions

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

#### Protocol conformance and refinements

Protocols should have the possibility to conform and refine other protocols containing throwing functions based on the subtype relationship of their functions. This way it would be possible to throw a more specialised error or don't throw an error at all.

```swift
protocol Throwing {
    func f() throws
}

struct ConcreteNotThrowing: Throwing {
    func f() { } // okay, doesn't have to throw  
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

struct ConcreteThrowingBlueError: ThrowingColoredError {
    func f() throws(BlueError) { ... } // okay, subtype
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

#### Override checking

A declaration in a subclass that overrides a superclass declaration can be a subtype of the superclass declaration, for example:

```swift
class Superclass {
  func f() throws { }
  func g() throws(ColoredError) { }
}

class Subclass: Superclass {
  override func f() throws(ColoredError) { } // okay
  override func g() throws(BlueError) { }   // okay
}

class Subsubclass: Subclass {
  override func f() { } // okay
  override func g() throws(DeepBlueError) { }  // okay
}
```

### Type inference

The type checker can infer thrown error types in a number of different places, making it easier to carry specific thrown type information through a program without additional annotation. This section covers the various ways in which thrown errors interact with type inference.

#### Closure thrown type inference

Function declarations must always explicitly specify whether they throw, optionally providing a specific thrown error type. For closures, whether they throw or not is inferred by the Swift compiler. Specifically, the Swift compiler looks at the structure of body of the closure. If the body of the closure contains a throwing site (either a `throw` statement or a `try` expression) that is not within an exhaustive `do...catch`  (i.e., one that has an unconditional `catch` clause), then the closure is inferred to be `throws`. Otherwise, it is non-throwing. Here are some examples:

```swift
{ throw E() } // throws

{ try call() } // throws

{ 
  do {
    try call()
  } catch let e as CatError {
    // ...
  }
} // throws, the do...catch is not exhaustive

{ 
  do {
    try call()
  } catch e {}
    // ...
  }
} // does not throw, the do...catch is exhaustive
```

With typed throws, the closure type could be inferred to have a typed error by considering all of the throwing sites that aren't caught (let each have a thrown type `Ei`) and then inferring the closure's thrown error type to be `errorUnion(E1, E2, ... EN)`. 

> **Swift 6**: This inference rule will change the thrown error types of existing closures that throw concrete types. For example, the following closure:
>
> ```swift
> { 
>   if Int.random(in: 0..<24) < 20 {
>     throw CatError.asleep
>   }
> }
> ```
>
> will currently be inferred as `throws`. With the rule specified here, it will be inferred as `throws(CatError)`. This could break some code that depends on the precisely inferred type. To prevent this from becoming a source compatibility problem, we apply a rule similar to the one for `do...catch` statements to limit inference: this new closure inference rule only applies in Swift 5 code when there is at least one `try` within the closure body. This way, one can only infer a more specific thrown error type in a closure when one of the called functions has specified a thrown error type.
>
> Note that one can explicitly specify the thrown error type of a closure to disable this type inference, which has the nice effect of also providing a contextual type for throw statements:
>
> ```swift
> { () throws(CatError) in
>   if Int.random(in: 0..<24) < 20 {
>     throw .asleep
>   }
> }
> ```

#### Associated type inference

An associated type can be used as the thrown error type in other protocol requirements. For example:

```swift
protocol CatFeeder {
    associatedtype FeedError: Error 
    
    func feedCat() throws(FeedError) -> CatStatus
}
```

When a concrete type conforms to such a protocol, the associated type can be inferred from the declarations that satisfy requirements that mention the associated type in a typed throws clause. For the purposes of this inference, a non-throwing function has `Never` as its error type and an untyped `throws` function has `any Error` as its error type. For example:

```swift
struct Tabby: CatFeeder {
  func feedCat() throws(CatError) -> CatStatus { ... } // okay, FeedError is inferred to CatError
}

struct Sphynx: CatFeeder {
  func feedCat() throws -> CatStatus { ... } // okay, FeedError is inferred to any Error
}

struct Ragdoll: CatFeeder {
  func feedCat() -> CatStatus { ... } // okay, FeedError is inferred to Never
}
```

#### `Error` requirement inference

When a function signature uses a generic parameter or associated type as a thrown type, that generic parameter or associated type is implicitly inferred to conform to the `Error` type. For example, given this declaration:

```swift
func f<E>(e: E) throws(E) { ... }
```

the function `f` has an inferred requirement `E: Error`. 

### Standard library adoption

There are a number of places in the standard library where the adoption of typed throws will help maintain thrown types through user code. This section details those changes to the standard library:

#### Converting between `throws` and `Result`

`Result`'s [init(catching:)](https://developer.apple.com/documentation/swift/result/3139399-init) operation translates a throwing closure into a `Result` instance. It's currently defined only when the `Failure` type is `any Error`, i.e.,

```swift
init(catching body: () throws -> Success) where Failure == any Error { ... }
```

Replace this with an initializer that uses typed throws:

```swift
init(catching body: () throws(Failure) -> Success)
```

The new initializer is more flexible: in addition to retaining the error type from typed throws, it also supports non-throwing closure arguments by inferring `Failure` to be equal to `Never`.

Additionally, `Result`'s `get()` operation:

```swift
func get() throws -> Success
```

should use `Failure` as the thrown error type:

```swift
func get() throws(Failure) -> Success
```

#### `Task` creation and completion

The [`Task`](https://developer.apple.com/documentation/swift/task) APIs have a `Failure` type similarly to `Result`, but use a pattern of overloading on `Failure == any Error` and `Failure == Never` to handling throwing and non-throwing versions. For example, the `Task` initializer is defined as the following overloaded pair:

```swift
init(priority: TaskPriority?, operation: () async -> Success) where Failure == Never
init(priority: TaskPriority?, operation: () async throws -> Success) where Failure == any Error
```

These two initializers can be replaced with a single initializer using typed throws:

```swift
init(priority: TaskPriority?, operation: () async throws(Failure) -> Success)
```

The result is both more expressive (maintaining typed error information) and simpler (because a single initializer suffices). The same transformation can be applied to the `detached` function that creates detached threads, where the two overloads are replaced with the following:

```swift
@discardableResult
static func detached(
    priority: TaskPriority? = nil,
    operation: @escaping () async throws(Failure) -> Success
) -> Task<Success, Failure>
```

Finally, the `value` property of `Task` is similarly overloaded:

```swift
extension Task where Failure == Never {}
  var value: Success { get async }
}
extension Task where Failure == any Error {
  var value: Success { get async throws }
}
```

These two can be replaced with a single property:

```swift
var value: Success { get async throws(Failure) }
```

#### `AsyncIteratorProtocol` associated type

`AsyncSequence` iterators can throw during iteration, as described by the `throws` on the `next()` operation on async iterators:

```swift
public protocol AsyncIteratorProtocol {
  associatedtype Element
  mutating func next() async throws -> Element?
}
```

Introduce a new associated type `Failure` into this protocol to use as the thrown error type of `next()`, i.e.,

```swift
associatedtype Failure: Error = any Error
mutating func next() async throws(Failure) -> Element?
```

Then introduce an associated type `Failure` into `AsyncSequence` that provides a more convenient name for this type, i.e.,

```swift
associatedtype Failure where AsyncIterator.Failure == Failure
```

With the new `Failure` associated type, async sequences can be composed without losing information about whether (and what kind) of errors they throw.

With the new `Failure` type in place, we can adopt [primary asociated types](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md) for these protocols:

```swift
public protocol AsyncIteratorProtocol<Element, Failure> {
  associatedtype Element
  associatedtype Failure: Error = any Error
  mutating func next() async throws(Failure) -> Element?
}

public protocol AsyncSequence<Element, Failure> {
  associatedtype AsyncIterator: AsyncIteratorProtocol
  associatedtype Element where AsyncIterator.Element == Element
  associatedtype Failure where AsyncIterator.Failure == Failure
  __consuming func makeAsyncIterator() -> AsyncIterator
}
```

This allows the use of `AsyncSequence` with both opaque types (`some AsyncSequence<String, any Error>`) and existential types (`any AsyncSequence<Image, NetworkError>`). 

#### Operations that `rethrow`

The standard library contains a large number of operations that `rethrow`. In all cases, the standard library will only throw from a call to one of the closure arguments: it will never substitute a different thrown error type. Therefore, update every `rethrows` function in the standard library to carry the thrown error type from the closure parameter to the result, i.e., the optional `map` operation will be change from:

```swift 
public func map<U>(
  _ transform: (Wrapped) throws -> U
) rethrows -> U?
```

to

```swift
public func map<U, E>(
  _ transform: (Wrapped) throws(E) -> U
) rethrows(E) -> U?
```

## Source compatibility

This proposal has called out three specific places where the introduction of typed throws into the language will affect source compatiblity. In each place, a minimal source-breaking aspect of the change has been separated out so that it will be enabled only in the next major language version (Swift 6), and Swift 5 has these additional limitations:

* For `do...catch`, the caught error type will be `any Error` if there are no `try` expressions.
* For `rethrows`, untyped closure parameters are treated as throwing `any Error`
* For closure thrown error type inference, the thrown error type will be `any Error` if there are no `try` expressions.

To make use of the full typed throws semantics in Swift 5, developers can enable the [upcoming feature flag](https://github.com/apple/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md) named `FullTypedThrows`. 

Note that the source compatibility arguments in this proposal are there to ensure that Swift code that does not use typed throws will continue to work in the same way it always has. Once a function adopts typed throws, the effect of typed throws can then ripple to its callers.

## Effect on ABI stability

The ABI between an function with an untyped throws and one that uses typed throws will be different, so that typed throws can benefit from knowing the precise type. For most of the standard library changes, an actual ABI break can be avoided because the implementations can make use of [`@backDeploy`](https://github.com/apple/swift-evolution/blob/main/proposals/0376-function-back-deployment.md). However, the suggested change to `AsyncIteratorProtocol` might not be able to be made in a manner that does not break ABI stability.

## Effect on API resilience

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

- If you think you don't know what will happen in the future and breaking changes should be avoided as much as possible, then just throw `any Error`. But keep in mind that you are less explicit about what can happen and also take the possibility for the API user to rely on the existence of the error (by using the compiler) leading to issues mentioned in [Motivation](#motivation).

- If you provide an extension point to your API like a protocol (e.g. a `DataLoader` like above) that can be used to customize the behaviour of your API, then try to omit forcing specific errors on the API user. Most of the time you as an extension point provider just want to know that something went wrong. If you need multiple cases of errors then keep the amount as small as possible and eventually do compatibility converting on the API developer side outside of the extension point implementation. "Only ask for what you need" applies here.

## Alternatives considered

### Multiple thrown error types

This proposal specifies that a function may throw at most one error type, and if there is any reason to throw more than one error type, one should use `any Error` (or the equivalent untyped `throws` spelling). It would be possible to support multiple error types, e.g.,

```swift
func fetchData() throws(FileSystemError, NetworkError) -> Data
```

However, this change would introduce a significant amount of complexity in the type system, because everywhere that deals with thrown errors would have to deal with an arbitrary set of thrown errors.

A more reasonable direction to support this use case would be to introduce a form of anonymous enum (often called a *sum* type) into the language itself, where the type `A | B` can be either an `A` or ` B`. With such a feature in place, one could express the function above as:

```swift
func fetchData() throws(FileSystemError | NetworkError) -> Data
```

Trying to introduce multiple thrown error types directly into the language would introduce nearly all of the complexity of sum types, but without the generality, so this proposal only considers a single thrown error type.

### Treat all uninhabited thrown error types as nonthrowing

This proposal specifies that a function type whose thrown error type is `Never` is equivalent to a function type that does not throw. This rule could be generalized from `Never` to any *uninhabited* type, i.e., any type for which we can structurally determine that there is no runtime value. The simplest uninhabited type is a frozen enum with no cases, which is how `Never` itself is defined:

```swift
@frozen public enum Never {}
```

However, there are other forms of uninhabited type: a `struct` or `class` with a stored property of uninhabited type is uninhabited, as is an enum where all cases have an associated value containing an uninhabited type (a generalization of the "no cases" rule mentioned above). This can happen generically. For example, a simple `Pair` struct:

```swift
struct Pair<First, Second> {
  var first: First
  var second Second
}
```

will be uninhabited when either `First` or `Second` is uninhabited. The `Either` enum will be uninhabited when both of its generic arguments are uninhabited. `Optional` is never uninhabited, because it's always possible to create a `nil` value.

It is possible to generalize the rule about non-throwing function types to consider any function type with an uninhabited thrown error type to be equivalent to a non-throwing function type (all other things remaining equal). However, we do not do so due to implementation concerns: the check for a type being uninhabited is nontrivial, requiring one to walk all of the storage of the type, and (in the presence of indirect enum cases and reference types) is recursive, making it a potentially expensive computation.  Crucially, this computation will need to be performed at runtime, to produce proper function type metadata within generic functions:

```swift
func f<E: Error>(_: E.Type)) {
  typealias Fn = () throws(E) -> Void
  let meta = Fn.self
}

f(Never.self)                // Fn should be equivalent to () -> Void
f(Either<Never, Never>.self) // Fn should be equivalent to () -> Void
f(Pair<Never, Int>.self)     // Fn should be equivalent to () -> Void
```

The runtime computation of "uninhabited" therefore carries significant cost in terms of the metadata required (one may need to walk all of the storage of the type) as well as the execution time to evaluate that metadata during runtime type formation. Therefore, we stick with the much simpler rule where `Never` is the only uninhabited type considered to be special.
