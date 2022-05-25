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

#### Integers

Default integer type is an `Integer` type which is big integer - i.e. has no constraints about possible range. This type is primarly used by the compile-time functions and by compiler built-in types. All numeric literals have type `Integer`.

Type `Int` is a type based on the `Integer` type that have constraints on possible range of values. To be clear, `Int` is a function that takes a number and returns a type, for example `Int(257)` is an integer that accept numbers in range -2^256..2^256 which is corresponds to the `int257` type from FunC or TVM. It is also possible to declare integers with different ranges, for example `Int(64)` which accept integers in range -2^63..2^63, etc.

Since `Int` is a struct type it cannot be create directly from the literal (can be changed in the future). To create value of type `Int(257)`, you need to call method `new`, for example `Int(257).new(10)`.

Note that types with different range of values cannot be used simultaneously - they have different types, i.e. `Integer` ≠ `Int(257)` ≠ `Int(64)`.

This separation between `Integer` and `Int` was made due to following reasons:
1. Tact have compile-time functions that may want to operate on the different range of types than `Int257` from TVM.
2. We want to add custom-ranged integer types.

#### Cell

`Cell` is a built-in type. It has no methods currently. `Cell` was made from `Builder` by `builder.build()`.

`Builder` is a built-in type. It have few methods:
1. `Builder.new()` method will create an empty builder. It is analogue to the `NEWC` instruction.
2. `builder.store_int(value, bits)` method will add value of type `Integer` to the builder. It is analogue to the `STIQ` instruction. Note that if you want to store value with `Int` type you need to use other methods.
3. `builder.build()` method will create a `Cell` from the builder. It is analogue to the `ENDC` instruction.

#### Tuples?

TBD?


### Standard types

TBD: what do we ship as a standard library: ranges, subranges of integers, Option/Result etc?



## Structs

TBD: define structs without introducing generics first. Generics will be dealt in a separate section.

Structs (aka "product types") allow  multiple fields of different types that exists as one entity. Product types have fields and methods — functions that operate on these fields.

Example of the declaration of the struct:
```
let Foo = struct {
  // val keyword is required to declare fields
  val field1: Type1
  val field2: Type2

  // Free method that can be called as Foo.free_methods()
  fn free_method() { /* implementation */ }
  // Method that can be called only on Foo instance
  fn method(self: Self) { /* implementation */ }
};
```

or using sugared version:
```
struct Foo {
  /* fields and methods */
}
```

It is also possible to generalize structs using compile-time functions. To learn more, see [Generics section](#generics).

Usage of the struct types:
```
// Type construction
let foo = Foo{field1: value1, field: value2, /* etc */};

// Type deconstruction (isn't supported by the compiler, can be changed in the future):
let Foo{field1: binding_name1, field2: binding_name2} = foo;
let Foo{field1: binding_name1, _} = foo;

// Get acces to the field 
let _ = foo.field1;

// Call free method
let _ = Foo.free_method();
// Call method
let _ = foo.method();
```

### Unions

TBD: 
* define unions without introducing generics first.
* explain newtype semantics and lack of newtypes for cases
* show match statement

**WARN**: The below is outdated.

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

Functions in Tact is like functions in other programming languages. They have list of typed arguments and body. When function is called it returns some value or do something.

Declaration of the function is as follows:
```
let func = fn(arg1: Type1, arg2: Type2, ...) -> Type {
  /* body of stmts */
}
```
or using sugared version:
```
fn func(arg1: Type1, arg2: Type2, ...) -> Type {
  /* body of stmts */
}
```

It is also possible to generalize functions using compile-time functions. To learn more, see [Generics section](#generics).

It is possible to create anonymous functions, i.e. function without names and use them in place as in follows:
```
let two = (fn(x: Int) { x + 1 })(1);
```

### Interfaces

TBD: why we need interfaces and their semantics.






## Generics

Generic programming is a significant part of modern programming languages. It is important concept without which it is impossible to imagine programming. 

Many other languages, such as C#, Java, Kotlin, Rust creates their own metalanguages with special syntax and semantic and builds generic programming based on these metalanguages. It is common concept that allow to simply integrate generic programming in the common programming process, but this way has limitations (FIXME: better word) benefits:

1. Metalanguage is yet another language with its own rules and syntax which complicates already complicated language.
2. Supporting two languages takes more time than supporting one language.
3. Metalanguage forces to duplicate concepts within a language which generates an incostistency.
4. It is hard to implement such conceptions in metalanguage that we want to see in Tact, such as custom ranges, etc.

Due to above we decide do not introduce special metalanguage to Tact. Our system based on one language with compile-time functions operating on the types which is just values like other values. Our system has much in common with Zig concepts.

TODO: add section about justification that generics split languages at two different.

### Compile-time evaluation

As we described above, we do not to split language at two different, so we provide compile-time evaluation of functions that written in the Tact language. General goal of the compile-time functions is provide a way to generate types, functions and change them. It provides powerful possibilities to create different type constructions like templates in C++ or comptime in Zig.

There are some limitations which functions can be evaluated in the compile-time:
1. Compile-time functions cannot call asm bytecode.
2. Compile-time functions cannot access to the runtime-only variables and constants, such as contract balance, chain gas fee, etc.
3. Compile-time functions cannot call other runtime-only functions.

Any function that satisifes above rules can be evaluated in compile-time.

Main rule of the compile-time: **everything can be evaluated must be evaluated**. This rule is called "global evaluation rule" (TBD?) This means that compiler will try to evaluate everything can be evaluated using common with runtime rules of execution functions (except runtime-only variables and functions).

Let's understand this principle. Suppose you have following code:
```
// Definitions
let CONST = 10;
fn const_fn() { 10 }
fn add(left: Integer, right: Integer) { left + right }
fn get_balance() -> Integer { /* suppose this function return balance of the contract */ }

// Usage
fn do_stuff() {
  let a = add(CONST, const_fn());
  let b = get_balance();
  let c = add(b, 10);
}
```

Let's consider these statements:
1. `CONST` and `const_fn` is constants which returns constant values. They will be evaluating everywhere.
2. `add` is a function that accept arguments and do not access runtime variables, so it _can be_ evaluated.
3. `get_balance` is a function that access runtime variable (balance), so it cannot be evaluated in runtime.
4. `let a = add(CONST, const_fn());` is evaluable expression due to it call constant function `const_fn` and function that can be evaluated (`add`). So it will be evaluated in compile-time and this expression will be rewritten to the `let a = 20` expr whih is a constant.
5. `let b = get_balance();` is a runtime expression because of `get_balance` is a runtime function, so this expression won't be evaluated.
6. `let c = add(b, 10)` is a runtime expression due to it access runtime variable `b`, so this expression won't be evaluated.

This evaluating concepts have significant benefit in our system: compile-time-only functions will be evaluated in compile-time and will not get to runtime. This allow to build type system based on the compile-time functions. To illustrate usage of this let's consider following code fragment (syntax can be changed in future):

```
fn IntRanged(from: Integer, to: Integer) -> Type {
  struct {
    val value: Integer

    fn new(value: Integer) -> Self {
      if (value < from || value > to) {
        throw_exception("Value out of range")
      } else {
        Self { value: value }
      }
    }
  }
}
let Int10to20 = IntRanged(10, 20);
let fifteen = Int10to20.new(15); // OK
let ten = Int10to20.new(10); // OK
let nine = Int10to20.new(9); // ERR: Value out of range
```

Due to compile-time functions, you can construct types inside of functions like above. This is powerful thing that allow to express everything you can express in other programming languages and even more!

### Higher-order types

TBD: What is "Type", "Interface", "Union", "Struct" etc. and their APIs

### Generic types

Generics is a construct that allow operate on values without knowing of concrete type of the value. Generics usually allow to describe interface of a type of a value, and use this interface inside of generic function or generic type. Generics in Tact is implemented using compile-time functions. Due to global evaluation rule generic functions will be instansiated in place of usage.

Let's see example of generic struct definition:
```
fn Pair(T: Type) -> Type {
  struct {
    val value1: T
    val value2: T

    fn new(val1: T, val2: T) -> Self {
      struct {
        value1: val1,
        value2: val2
      }
    }
  }
}

// This function call will be evaluated in compile-time
let PairOfInts = Pair(Integer);

let data = PairsOfInt.new(10, 20); // OK
let data = PairsOfInt.new(
    Builder.new(), Builder.new()
  ); // ERR: Expected `Integer` type, got `Builder` type
```

TBD: constraints for generics.

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



