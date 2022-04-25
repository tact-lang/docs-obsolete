# Tact Language Specification

## Base types

What are TVM types and what are Tact types; what are the mapping from one to another?

* numbers 
* strings
* tuples
* tensors - do we even need them; for ABI we need stable distinction between tuples/tensors?
* cells

## Algebraic types

* structs
* enums
* unions

## Serialization strategies

* tuples vs cell trees?
* automatic VS annotated VS custom serializers

## Namespaces and visibility

We could do Rust-style namespaces: stuff from `foo.tact` can be imported as:

```
use foo;   // refer later to foo::T

use foo::T // refer later to T directly
```

Visibility via explicit `pub` declaration.


## Actor interfaces

### Typed message bodies

**Option A:** arbitrary message signatures:

```
foo(arg1: T1, arg2: T2) { ... }
```

**Option B:** fixed argument `body` with user-specified type:

```
foo(body: T) { ... }
```

Option A is more typical for API design, but works poorly with system-provided context (value, sender, extra currencies).

Also option B means there's declaration of `T` somewhere with its custom serialization definition (if needed).

### Internal/External notation

**Option A:** explicit `external`/`internal` keywords (or `ext`/`int` :-):

```
external transfer(body: Message) { ... }

internal process(value: Coins, body: Message) { ... }
```

Naming is a bit off as "internal" also seems like "scope-private" to newcomers.

**Option B:** uniform declaration, but distinguish internal/external by must-have params:

Internal messages must have `sender` and/or `value` argument. Also: do we allow system args be skipped?

```
handler transfer(body: Message) { ... } // external

handler process(sender: Address, value: Coins, body: Message) { ... } // internal
```

