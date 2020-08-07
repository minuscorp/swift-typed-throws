
## General motivation

???

- multiple layers, public apis ???

## Error scenarios considered

### Scenario 1: Specific thrown error, general catch clause

```
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

### Scenario 2: Specific thrown error, specific catch clause

```
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

### Scenario 3: Specific thrown error, multiple catch clauses

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

### Scenario 4: Unspecific thrown error

- Current behaviour of Swift applies

## Type erasure

Erasing an error type of a function that throws is easy as

```
catch {
    // assume the error is inferred as `CatError`
    let typeErasedError: Error = error
}
```

## `rethrow` (generic errors in map, filter etc)

???

## ABI stability

???

## Error structure

???

## `async` and `throws`

???

## Equivalence between `throws` and `Result`

??? Maybe this is for another proposal