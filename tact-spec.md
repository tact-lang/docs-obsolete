# Tact Language Specification

Tact types are modelled as _sets of possible values_. Therefore compatible type for assignment is a subset of values.

Each new defined type is strictly separate and requires explicitly conversion. E.g. `Bool` is not interchangeable with integers 0 and -1.

## Base types

What are TVM types and what are Tact types; what are the mapping from one to another?

### Never

The type that has no possible values. Used for eliminating branches of code and values.

```
type Never = builtin::Never;
```

### Null

```
type Null = builtin::NULL;
```

### Bool

```
type Bool = 0 | -1;
```

### Int257

Base integer type.

```
type Int257 = builtin::Int257;
```

### Range

TBD.

### tuples

TBD.

### tensors

Do we even need them; for ABI we need stable distinction between tuples/tensors?

### Cell

Generic and raw cell


## New types

### New type declaration

```
type T = <type-returning expression>;
```

### Structs

Product types:

```
struct T {
  a: A,
  b: B,
  ...
}
```

or: 

```
type T = struct { ... }
```

The latter might be useful in some generic type-building contexts.

### Enums

Sum types:

```
enum T {
   A,
   B(T2),
}
```

We may consider not having enums in favor of unions. Or not having unions at all.

### Unions

Fun idea: maybe merge unions and ranges?

So that `0..<10` is shorthand for `0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9`.


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

### Message tag

Each message in an actor must be uniquely identified by a 32-bit tag. 
By default, 32-bit message tag is calculated as CRC32 checksum of the method declaration.

The message tag can be overriden (for compatibility or disambiguation) by explicit declaration:

```  
  internal(1008) my_message(...) { ... }
```

Pros: simple, elegant.

Cons: might be at odds with other attributes if there are any.

Alternative:

```
  @message_tag(1008)
  internal my_message(...) { ... }
```


