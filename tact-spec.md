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


## Phases

Tact execution consists of two phases: _comptime_ and _runtime_.

_Comptime Tact_ is a smaller language that does not have interfaces, actors and generics. 
It has immutable bindings, values of type "Type" and set-operations on types.

The goal of _Comptime Tact_ is to generate types, functions and impls that will be used in runtime.

Tact separates these phases syntactically:

1. File-level code is comptime: let bindings, type declarations, function declarations.
2. Comptime function body: comptime.
3. Runtime function body: runtime.
4. Struct/enum body: comptime.
5. Impl body: comptime.
6. Interface body: comptime.



## New types

## User-defined types
User-defined types is a types which is defined by an used based on already existing types. 

### Product types
Product types is a types that can have multiple fields of different types that exists as one entity. Besides of fields product types can also have functions that operates on type fields TODO.

Example of the declaration of the product type:
```
let Foo = type {
  field1: Type1,
  field2: Type2,
  // etc
};
```

Type can be generalized using function:
```
let Foo = fn(X: Type) -> Type {
  type {
    field1: X
  }
};
// Usage
let IntFoo = Foo(Int257);
```

or using shortcut syntax:
```
let Foo(X: Type) = type {
  field1: X
};
// Usage
let IntFoo = Foo(Int257);
```

Type construction:
```
let foo = Foo{field1: value1, field: value2, /* etc */};
```

Type deconstruction:
```
let Foo{field1: binding_name1, field2: binding_name2} = foo;
let Foo{field1: binding_name1, _} = foo;
```

Get access to the field:
```
let _ = foo.field1;
```

Mutate field: 
```
~foo.field = value;
```

### Unions
Union is a sum-type means it is a type that can be one of multiple possible options. Union type does not create new constructors.

Union type definition:
```
let Red = type{ /* possible fields */ };
let Green = type{ /* possible fields */ };
let Color = Red | Green;
```

Union can be generalized using function:
```
let Color = fn(X: Type) -> Type {
  Red(X) | Green(X)
};
// Usage
let IntColor = Color(int257);
```

or using shortcut syntax:
```
let Color(X: Type) = Red(X) | Green(X);
// Usage
let IntColor = Color(Int257);
```

Union construction:
```
fn accept_color(color: Color) {}
fn do_something() {
  // Construction
  let color = Red{ /*fields*/ };
  // Usage
  accept_color(color);
}
```

Union deconstruction:
```
let data = match color {
  Red{value} => value,
  Green{value} => value,
};
```

Union can be anonymous:
```
let Red = type{};
let Green = type{};
fn returns_color() -> Red | Green { ... }
```

## Memory representation
Data structures can be represented in the TVM bytecode as `Cell`s or `Tuple`s.

### Representation using Cells
Cells allow to store arbitrary sequence of bytes up to 1023 bits. 

Pros of using Cells:
1. No need of deserializing when sending structure using `SENDRAWMSG` or when storing in the storage `c4`.
2. Compact storage of integers that do not cover the entire `int257` range.

Cons of using Cells:
1. Necessity to create hierarchical cell structures because data the structure can be larger than 1023 bytes.
2. Expensive read operations (compared to tuples).
3. No write operations.

### Representation using Tuples
Tuple in TVM is a list of values with arbitrary types up to 255 elements.

Pros of using Tuples:
1. Simple read/write of arbitrary elements.

Cons of using Tuples:
1. Necessity to deserialize into Cells when sending structure using `SENDRAWMSG` or when storing in the storage `c4`.
2. Expensive operations with tuples having more than 16 elements.

### Solution: Always use tuples
Cells are designed for sending messages between actors and storing values in persistent storage `c4`. Instead, tuples are designed to represent algebraic data types (see 1.1.3 section of [TVM paper](https://ton-blockchain.github.io/docs/tvm.pdf)).

Tuples have more simpler API which allows to get values from the tuple using 1 bytecode operation `INDEX` while extraction from cell require:
1. Create slice if not created.
2. Push bits to be skipped at the stack.
3. Skip bits using `SDSKIPFIRST`.
4. Read field.

Tuple elements can be mutated using `SETINDEX` opcode while slice of cell can be mutated by:
1. Create new builder.
2. Store all previous data before field need to be mutated.
3. Store mutated field.
4. Store all other fields.

Based on the above it is better to use tuples to represent data structures in the TVM.

Suggestions:
1. Warning lint when use struct definitions with more than 16 fields.

Unresolved questions:
1. Should memory representation has stable rules or be a implement-details?
2. Should we introduce Cell memory representation for messages or storage elements?

#### Structs as tuples
TODO

#### Sum types as tuples
Sum types can be stored like a struct but with addition 0-index field representing a discriminant.

## Serialization strategies
* automatic VS annotated VS custom serializers

## Namespaces and visibility

We could do Rust-style namespaces: stuff from `foo.tact` can be imported as:

```
use foo;   // refer later to foo::T

use foo::T // refer later to T directly
```

Visibility via explicit `pub` declaration.


## Actors

### Interfaces and impls

We do need to support impls for multiple interfaces, similar to trait impls in Rust:

```
interface A {
  internal foo();
}
interface B {
  internal bar();
}
impl A for T {
  internal foo() { ... };
}
impl B for T {
  internal bar() { ... };
}
```

For ordinary contracts with their own interface, do we have a shortcut to avoid duplicate definitions?

```
actor T {
  internal something() { ... };
}
```

Maybe use `actor` instead of `impl` for interface impls? 

Would we have traits for plain structs and methods?

See also https://www.swiftbysundell.com/articles/swift-actors/

### Message send syntax

Ordinary functions/methods are easy and Rust/Swift-style:

```
T::foo(a,b,c);
a.foo(b,c);
```

We might need to distinguish message-sending syntax from ordinary function calls to:
* quickly see if the function is pure or not;
* avoid possible ambiguities

No special syntax:

```
addr.msg(args);
```

Send keyword:

```
send addr.msg(args);
```

Fancy punctuation:

```
@addr.msg(args);
addr->msg(args);
addr~msg(args);
msg(args)->addr;
```

Bureaucratic:

```
send(addr, msg, args);
addr.send(msg, args);
addr.send(Message{value: 0, body: ...});
```

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


