# Tact Design

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
  * [Compile-time reflection](#compile-time-reflection)
  * [Dependent types](#dependent-types)
  * [Higher-order types](#higher-order-types)
  * [Generic types](#generic-types)
  * [Generic functions](#generic-functions)
  * [Annotations](#annotations)
  * [Numerical bounds](#numerical-bounds)
* [Representation in TVM](#representation-in-tvm)
  * [Memory Representation](#memory-representation)
* [Mutability](#mutability)
* [Serialization](#serialization)
* [Namespaces and visibility](#namespaces-and-visibility)
* [Operators](#operators)
* [Control flow](#control-flow)
* [Actors](#actors)


## Why Tact?

Blockchains are naturally hard to scale and make tradeoffs between decentralization and expressiveness. 
TON offers a different tradeoff: actor-based architecture allows both decentralization and infinite scalability, 
but developers have to write complex multi-actor apps with asynchronous communication.

Fift and FunC provide precise control over execution and require developer to keep in mind important aspects of cross-actor communication. 
To unleash the full power of TON architecture we need a higher-level language designed specifically for cross-actor communication. 
With rich type system, automatic verification of complex invariants and assurances of correct gas usage across multiple actors in the system.

## Tact features

**Actor-oriented:** Tact is designed specifically for the TON actor model. Strongly typed messages enforce communication contracts between actors.

**High-level type system with algebraic types and numeric bounds** allows expressing precise invariants that can be checked statically.
Compiler guides developer to put runtime checks where necessary to make the types match across actor and call boundaries.

Tact offers **automatic serialization into cells** and **partial access to cells** to maximize efficiency while letting the developer focus on the problem at hand.

**Gas control:** Tact makes cross-contract messages safe with precise gas commitments and compiler checks of the execution costs. 
Variable costs either have static bounds, or checked explicitly in runtime.


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

fn plus_one(a: Int(8)) -> Int(257) {
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

Union (aka sum-type) means it is a type that can be one of multiple possible options. Difference between unions (in Tact) and tagged unions (in Rust, Haskell, Scala) is that any variant in tagged unions must be created using constructor, while in unions there are no constructors at all. Unions is a structural typing type-sum while tagged union is nominative typing type-sum.

Union type definition:
```
let UnionType = union {
  case Type1;
  case Type2;

  // Free method that can be called as UnionType.free_methods()
  fn free_method() { /* implementation */ }
  // Method that can be called only at UnionType instance
  fn method(self: Self) { /* implementation */ }
};
```

or using sugared version:

```
union UnionType {
  /* cases and methods */
}
```

It is also possible to generalize structs using compile-time functions. To learn more, see [Generics section](#generics).

When union was declared, also were declared cases - its possible variants. Case is type that is an possible option of an union. There are possible to use case when union was expected, but there are no way to use union when concrete type was expected.

```
struct Red{}
struct Green{val field: Type}
enum RG { case Red; case Green; }

// Create union instance
let x: RG = Red{};
// Match union instance
if (let x: Red = x) {
  // x has type Red
} else if (let Green {field} = x) {
  // variable `field` has type `Type`
}
```

#### Union subtyping

```
struct Red{}
struct Green{}
enum RG { case Red; case Green; }

// It is allowed here to pass Red, Green, RG types as argument but return value can have only `RG` type.
fn function(arg: RG) -> RG { arg }

let x: RG = Green{};
let test1: RG = function(Red{}); // OK.
let test2: RG = function(x); // OK.
let test3: Red = function(Red{}); // ERR: expected `RG` type, found `Red`.
```

#### Union extending
```
struct Red{}
struct Green{}
enum RG { case Red; case Green; }

// This:
enum RGB { case RG; case Blue; }
// is equivalent with this:
enum RGB { case Red; case Green; case Blue; }
```

#### Why not tagged unions?

When unions was discussed, all developers suggest that tagged unions has many ergonomic problems:
1. Hard to create and unpack.
2. Constructors is not types.
3. Make one tagged union from another isn't a simple operation.

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

### Compile-time reflection

TBD: get access to the state of the program in the compile-time and modify it.

### Dependent types

In tact it is need to support subset of dependent types, or rather dependent functions. It is flow from concept of compile-time functions, when return type is depend on the argument value. See following snippet:

```
fn id(T: Type) -> (fn(T) -> T) {
  fn(x: T) { x }
}
```

This function take value `T` of type `Type` and returns function with type `fn(T) -> T`. So, type of returned function depend on the incoming argument. It is need to express such constructions that must be provided due to fully-qualified generics support.

TBD: Do we need fully-qualified dependent types with dependent pairs? How them must be looks like?

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

Generics is a common programming pattern, so we provide some sugared version of generic definitions:
```
// for structs
let Foo(X: Type) = struct {
  val field: X
}
struct Foo(X: Type) {
  val field: X
}
// for functions
let add(X: Type) = fn(left: X, right: X) -> X {
  ...
}
fn add(X: Type)(left: X, right: X) -> X {
  ...
}
```

TBD: constraints for generics.

### Generic functions

TBD: functions and methods could be generic. 

### Annotations

TBD: syntax for annotations, why it's used.

### Numerical bounds

TBD: How numeric bounds work and why we need them: safe arithmetic, value/gas commitments.

## Representation in TVM

This section is about implement details of converting tact program to TVM bytecode.

### Memory representation

Data structures can be represented in the TVM bytecode as cells ro tuples

#### Representation using Cells
Cells allow to store arbitrary sequence of bytes up to 1023 bits. 

Pros of using Cells:
1. No need of deserializing when sending structure using `SENDRAWMSG` or when storing in the storage `c4`.
2. Compact storage of integers that do not cover the entire `int257` range.

Cons of using Cells:
1. Necessity to create hierarchical cell structures because data the structure can be larger than 1023 bytes.
2. Expensive read operations (compared to tuples).
3. No write operations.

#### Representation using Tuples
Tuple in TVM is a list of values with arbitrary types up to 255 elements.

Pros of using Tuples:
1. Simple read/write of arbitrary elements.

Cons of using Tuples:
1. Necessity to deserialize into Cells when sending structure using `SENDRAWMSG` or when storing in the storage `c4`.
2. Expensive operations with tuples having more than 16 elements.

#### Solution: Always use tuples
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

##### Struct as tuple
Struct has fields and methods. Field is member represent variable data, while method is a function that operates on this data.

It is possible to implement such "virtual table" like in typical OOP-languages such as C#, Java, but there are no need to do this in Tact. Tact isn't OOP language, so all methods can be converted to free functions, so there are no need to store methods next to struct instance.

In TVM values are stored on a stack separately. So, there are no problems with memory alignments and there is no difference between the order of fields in stack. Fields of struct stored in the same order as they declared.

Compiler can optimize memory representation. It can inline tuples as tensors if tuple is not used, or if this tuple used in other tuple. Compiler will always inline tuples consists of 0 or 1 element.

##### Union as tuple
Sum types can be stored like a struct but with addition 0-index field representing a discriminator.

TBD: Concrete memory model (see [discussion](https://github.com/tact-lang/docs/discussions/18)).

## Mutability

All variables are immutable by default. To declare mutable variable user should use special keyword (TBD: `let mut` or `var`).

Mutation occurs throught `<ident> = <expr>` syntax. TBD: `=`, `:=`, `<-` operator?

TBD: global mutable variables?

### Name shadowing

Shadowing of local variables is allowed. Shadowing of global variable (immutable or not) produce a warning.

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



