# Typed throws discussion

## General

Options we have developing the language while keeping compatibility: https://forums.swift.org/t/typed-throws/39660/66

So it seems there is no law that forbids *thinking* about small source adjustments. ;)

## `rethrows`

### Issue

> the key question with rethrows, which is whether the signature of A below is equivalent to that of B or that of C

```swift
// A
func foo(fn: () throws -> Void) rethrows

// B
func foo<erased E: Error>(fn: () throws E -> Void) rethrows E

// C
func foo<erased E: Error>(fn: () throws E -> Void) rethrows Error
```

see https://forums.swift.org/t/typed-throws/39660/53

### Solution 1

rethrow without converting (compatible to current)

```swift
func foo(_ bar: () throws /* Error */ -> ()) rethrows /* Error */ {
    try bar()
}
```

rethrow with converting (compatible to current)

```swift
func foo(_ bar: () throws /* Error */ -> ()) rethrows /* Error */ {
    do {
        try bar()
    } catch {
        throw CustomError(base: error)
    }
}
```

rethrow without converting

```swift
func foo<E>(_ bar: () throws E -> ()) rethrows E {
    try bar()
}

// If every function implicitly threw Never, we could write
func foo<E>(_ bar: () throws E -> ()) throws E {
    try bar()
}
// which would semantically be the same
```

rethrow with converting

```swift
func foo<E>(_ bar: () throws E -> ()) rethrows CustomError {
    do {
        try bar()
    } catch {
        throw CustomError(base: error)
    }
}
```

see https://forums.swift.org/t/typed-throws/39660/73

Example combinations:

```
func foo(closure: () throws -> ()) rethrows
func foo(closure: () throws -> ()) rethrows IntError
func foo(closure: () throws SomeError -> ()) rethrows
func foo(closure: () throws SomeError -> ()) rethrows IntError
func foo<E: Error>(closure: () throws E -> ()) rethrows E
```

In general:

- `rethrows T`: Throw `T` if parameter throws
- `rethrows === (rethrows T where T === Error)`: Throw `Error` if parameter throws

## `throw` and type inference

### Issue

```swift
struct Foo: Error { ... }
struct Bar: Error { ... }
var throwers = [{ throw Foo() }] // Inferred as `Array<() throws -> ()>`, or `Array<() throws Foo -> ()>`?
throwers.append({ throw Bar() }) // Compiles today, error if we infer `throws Foo`
```

see https://forums.swift.org/t/typed-throws/39660/70

### Solution 1

> So don't infer `throws Foo` , and infer `throws Error` instead. This will preserve source compatibility.

https://forums.swift.org/t/typed-throws/39660/71

### Solution 2

> `throws` functions should never be inferred to be the typed throw but the base `throws` in collections (array, set), unless the type is specified as follows:

https://forums.swift.org/t/typed-throws/39660/72

### Solution 3

```swift
func foo<T>(_: () throws T -> ()) { ... }
let _ = { throw Foo.error } // inferred as '() throws -> ()'
foo({ throws Foo.error }) // inferred as '() throws Foo -> ()'
```

> It also seems fairly likely to me that the source break wouldn't be that large (unless there's a more common use-case I haven't thought of?). It might make sense to tighten up the inference behavior as a change in Swift 6, where the Swift 5 compatibility mode would continue to infer the throws type as Error.

see https://forums.swift.org/t/typed-throws/39660/77

### Solution 4

> I guess the cleanest non source compat solution would be, if we could update source in Swift 6 from `throw FooError()` to `throw FooError() as Error` for all `throw` that happen directly in a `do` block or in closure that has it's type inferred.

see https://forums.swift.org/t/typed-throws/39660/123?