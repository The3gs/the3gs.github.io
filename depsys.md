layout: page
title: "Dependently Typed Systems Language"
permalink: /depsys

# Notes for a dependently typed systems language

I wish I could give you a name for it, but in this I fail.

Ideas are welcome.

## Assumptions about you

I am assuming that you have a decent understanding of dependent types, and type theory in general.

I will be using a Rust style syntax for the pseudocode in this document.
If anything strays far from Rust, I will attempt to explan my reasoning.

## Types

I think for a usable systems language you need to have complete control over the layout of memory.
For this reason, I beleive that it is correct for the Kind of Types to be indexed by the size of the memory it occupies.
As such, Types of Types are indicated as `Type(n)` where `n` is a natural number.
If possible I think that variable sizes should be expressed using the same type of numbers as is typically used for programming, so likely it will be an equivalent to Rust's `usize`.

The final type system will probably also need to worry about alignment of types,
but every time I try to think about the math to make alignment work,
I lost track of my second and third braincell, so I will be ignoring that for now.

### Structural Typing

Types are going to be structural. I think that this is best for a systems language as it allows flexibility in various things.

### Structs and Records

Unlike most similar languages, I think that it is correct to have multiple ways to package data together, as follows:

Structs are represented linearly in memory such that `struct {a: u8, b: u8, c: u8}` is identical to `struct {fido: u8, jimbo: struct {size: u8, age: u8}}`.
I will likely also include a way to transmute between such types freely.

Records are more similar to Rust's structs, where size and layout are decided by what will be the most efficient.

Mathematically, the two types are trivially isomorphic, but structs will be useful for many system tasks.

The size of a struct is the sum of the sizes of it's component ie if `T: Type(n)` and `U: Type(m)` then `struct {a0: T, a1: U}: Type(n + m)`

### enumerated types

I don't know what I want to call these, as I have never really loved any of the options, so I will just borrow enum from rust for now.

In dependent style, there should be options to have different constructors produce different types, but I am not certain on the syntax. It is WIP.

```
def Bool: Type(1) = enum {
  true,
  false,
}
```

### Functions

There are no Lambda expressions in the core language, though they can be recovered somewhat using syntax sugar.

Essentially, functions can only be defined at the top level.
This allows them to have a fixed size, as we are requiring.

The function pointers will be the same size as `usize`.

This also prevents partial application, but again we can work around this in some cases.

The syntax for a function pointer is `(a: A, b: B, ...) -> R`

Closures are of type `record { self_type :0 Type(_), call: (self: self_type, args...) -> R, self: self_type }`
where the size of the context is usually infered and can be abreviated as `(args...) => R`.

### Box Types

This is simple. It basically is a heap allocated value that allows you to fit a larger variable in a smaller one.

`T: Type(n) ⊦ Box(T): Type(PTR_SIZE)`

## Zero Quantified definitions

This is somthing that I think is useful that I am borrowing from Quantified Type Theory.
I only really have been exposed to this via a brief exploration with idris, so my knowledge is limited.

When a variable is declared using `:0` in stead of the usual `:` this means that the variable is only usable within types.

I think it might be needed to have a higher universe called Kind that is zero quantified, such that `Type(n):0 Kind`

## Recursion and Loops

I do not know if we should aim for a total language or not.
The language nerd in me says it should be total.
The systems programmer says we probably are going to need non-termination often enough to justify having our language inconsistent as a logic.

## Subtypes

I like subtyping. I would want subtyping of enums, and possibly some records, where elements can be trivially dropped.

Here are subtyping rules I have considered:
- `record {A: record {XYZ}, Xs} :> record {XYZ, Xs}` where XYZ doesn't have any names in common with Xs
  Note a special case where `record {A: record {}, Xs} :> record {Xs}`
- `T :> enum { default(T) }`
- `record {self: T, Xs} :> T` Note that `Xs` must all be dropable.
  This is probably most usful in expressing subtypes, as self can be a type, and Xs is ignorable in type annotations.

### Methods

A record with a self value can be used like a dynamic object as follows.

The print function is ficticious in the following example, and coerrces it's argument to a usize.
It also assumes that usize can be copied.

```

dec acc : record {
  self: usize,
  add: (usize, usize) -> usize,
  mul: (usize, usize) -> usize,
}

print(acc); // 0
acc.add(10);
print(acc); //10
acc.mul(11);
print(acc); //110

```

I think that there will be circumstances where you want the argument type to change, so the following is also valid.

```

dec door : record {
  open_door:0 Type(_),
  closed_door:0 Type(_),

  self: open_door,

  open: (closed_door) -> open_door,
  close: (open_door) -> closed_door
}

```

This allows state machines to be encoded in a type safe way, without requiring Rust style Zero-Sized-Types.

I quite like faiface's arrow syntax for calling functions in series, so I will likely have something like this as well.

```
def fibonacci(n: usize): usize {
  if n <= 1 {
    1
  } else {
    fibonacci(n - 1) + fibonacci(n - 2) // Might not work in final version. See recursion and loops
  }
}

def n: usize = 10;
n->fibonacci();
print(n) // 55
```

#### Special Methods

The following methods have special behavior
- `drop: (self : self_type) -> Unit`
  deletes the value
- `copy: (self : self_type) -> (self_type, self_type)`
  copies the value
- `call: (self : self_type, args...) -> R`
  allows calling the record similar to a function of type `(args...) -> R`


## Usage Examples

The following are example that I think woudl be at least close to programs in the final language.

### GPIO Pins
This is how I would expect a blinky example to look like on a Arduino or the like.
Don't worry too much about the IO type. It is mostly fictional for now.
```
use std::units::milliseconds;

def start(io: IO) {
  let led = io.pin(13);

  loop { // loop might not be in the language... See Recursion and loops
    led.on();
    io.delay(500->milliseconds());
    led.off();
    io.delay(500->milliseconds());
  }
}
```

## Other things

There are other things that I would like, but I do not know how to make them fit.

Here are some things I have stewed with:

- Some way of saying "This process never returns to the caller".
  Like Bottom in linear logic, and `?` in Par.
-  
