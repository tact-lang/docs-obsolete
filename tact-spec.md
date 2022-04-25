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

## Actor interfaces

* internal VS external handlers;
* message body types in handlers: flat list of arguments `f(a,b,c)` or body type `f(body: T)`;
 

### Typed message bodies

Option A: arbitrary message signatures:

```
foo(arg1: T1, arg2: T2) { ... }
```

Option B: fixed argument `body` with user-specified type:

```
foo(body: T) { ... }
```

Option A is more typical for API design, but works poorly with system-provided context (value, sender, extra currencies).

Also option B means there's declaration of `T` somewhere with its custom serialization definition (if needed).


