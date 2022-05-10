# Tact Language Specification

* [Why Tact?](#why-tact)
* [Syntax overview](#syntax-overview)
* [Type system](#type-system)
  * [Built-in types](#built-in-types)
  * [Standard types](#standard-types)
  * [Structs](#structs)
  * [Unions](#unions)
  * [Functions](#functions)
  * [Interfaces](#interfaces)
* [Generics](#generics)
  * [Compile-time execution](#compile-time-execution)
  * [Higher-order types](#higher-order-types)
  * [Generic types](#generic-types)
  * [Generic functions](#generic-functions)
  * [Annotations](#annotations)
  * [Numerical bounds](#numerical-bounds)
* [Representation in TVM](#representation-in-tvm)
* [Mutability](#mutability)
* [Serialization](#serialization)
* [Namespaces and visibility](#namespaces-and-visibility)
* [Operators](#operators)
* [Control flow](#control-flow)
* [Actors](#actors)


## Why Tact?

Tact is a safe and efficient high-level programming language for writing TON smart contracts.

Tact offers:

* High-level type system with algebraic types and numeric bounds.
* Automatic encoding to and from TON Cells with efficient partial access.
* First-class support for strictly-typed message handling.


## Syntax overview

TBD: short examples and key points
* variables
* code style
* comments
* semicolons
* types
* generics
* functions
* methods
* actors

```
#tact 1.0

let CONSTANT = 1;

fn plus_one(a: Int8) -> Int257 {
   return a + 1;
}

...
```

## Type system

TBD: quick overview of the main features/aspects.

* Strict type checking, no automatic casts.
* Generics supported via compile-time execution, where types are values.
* Algebraic types: sum types (unions) and product types (structs).
* Numeric bounds.


### Built-in types

TBD: relation between TVM types and Tact types.

#### Never

The type that has no possible values. Used for eliminating branches of code and values.

```
let Never = builtin::Never;
```

#### Null

```
let Null = builtin::NULL;
```

#### Int257

Base integer type.

```
type Int257 = builtin::Int257;
```

#### Cell

TBD.

#### Tuples?

TBD?


### Standard types

TBD: what do we ship as a standard library: ranges, subranges of integers, Option/Result etc?



## Structs

TBD: define structs without introducing generics first. Generics will be dealt in a separate section.

Structs (aka "product types") allow  multiple fields of different types that exists as one entity. Product types have fields and methods — functions that operate on these fields.

Example of the declaration of the product type:
```
let Foo = struct {
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
let Foo(X: Type) = struct {
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

TBD: 
* define unions without introducing generics first.
* explain newtype semantics and lack of newtypes for cases
* show match statement

Union (aka sum-type) means it is a type that can be one of multiple possible options. Union type does not create new constructors.

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

### Functions

TBD: how function declarations look like

TBD: what function type is

TBD: anonymous functions


### Interfaces

TBD: why we need interfaces and their semantics.






## Generics

TBD: one-liner about what generics are for and how Tact differs from other langs.

### Compile-time execution

TBD: How the program is executed in compile-time to build types.

Tact execution consists of two phases: _compile time_ (or _comptime_) and _runtime_.

The goal of _Comptime Tact_ is to generate types, functions and impls that will be used in runtime.


### Higher-order types

TBD: What is "Type", "Interface", "Union", "Struct" etc. and their APIs

### Generic types

TBD: how to make types generic; how stdlib generic types are defined.

### Generic functions

TBD: functions and methods could be generic. 

### Annotations

TBD: syntax for annotations, why it's used.

### Numerical bounds

TBD: How numeric bounds work and why we need them: safe arithmetic, value/gas commitments.

## Representation in TVM

Data structures can be represented in the TVM bytecode as cells and tuples.

TBD: describe when which typea are represented as cells or tuples. What compiler chooses, what 

## Mutability

All variables are immutable by default.

TBD: syntax for mutation of variables, and across methods.


## Serialization

TBD: encoding to/from Cells

TBD: partial decoding and updates

TBD: how to write custom serialization code

## Namespaces and visibility

TBD: describe what namespaces are and when they occur implicitly or explicitly.

TBD: how imports work


We could do Rust-style namespaces: stuff from `foo.tact` can be imported as:

```
use foo;   // refer later to foo::T

use foo::T // refer later to T directly
```

Visibility via explicit `pub` declaration.


## Operators

TBD: define built-in operators

NB: we want safe core operators, no operator overloading and no custom operators.

Since operators are hard-to-search punctuation, 
there should be little amount of those and they should be safe.

Overflowing/failing operations should be done via explicitly named functions.

## Control flow

TBD: conditionals

TBD: loops

TBD: errors

TBD: type checks (enumerating union members)


## Actors

TBD:

* how to declare actor interfaces
* internal/external
* message serialization
* sending messages
* strongly-typed sender
* gas commitments



