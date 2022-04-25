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
 


