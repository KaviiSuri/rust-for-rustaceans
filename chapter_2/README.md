# Chapter 2 - Types

Every rust [value](../chapter_1/#Value) has a type. Most fundamental role of a type is to tell how to interpret bytes.

## Alignment
Dictates where the bytes for a type can be stored. In practice, hardware imposes some constraints on how data is placed. 

All values must be *byte aligned*. This is because, otherwise, the CPU has to fetch extra bytes to read the value, even if it allows it.

Some values have more strict alignment rules. A 64-bit computer accesses memory in 8bit chunks (word size). To handle values over the boundaries of these chunks requires extra work.

Operation on data which is not aligned is called *misaligned access*.

Basic types in rust are generally aligned to their size (u16 is 2-byte-aligned etc.), while more complex types are aligned to the largest alignment of their member types.


## Layout
Memory representation of a value is called layout. Although the compiler gives very few guarantees about this, rust does have the `repr` attribute that can be used to enforce certain layouts.

e.g. `repr(C)` makes a type aligned like it would be in C, thus making it easier to interface with C.

`repr(transparent)` can be used only on single field types and guarantees that the layout of the outer type is exactly same as that of the inner type.

## Dynamically Sized Types and Wide Pointers
The `Sized` trait signifies that the size is known at compiletime. Most objects in rust are sized automatically, except in 2 cases:
1. trait objects
2. slices

These are *Dynamically Sized Types* or DSTs for short. But this can be a problem, as almost everything needs to be `Sized`, so much so that `T: Sized` is bound that is implicitly applied unless you do `T: ?Sized`.

The solution is to place them behind a `Wide Pointer` or `Fat Pointer`. It's just a normal pointer but also includes an extra word-size field that gives the additional information about the pointer that compiler needs to generate code.

For a slice, this info may be the length of the slice, for trait object, it can be more complicated. 

> A `Wide Pointer` is `Sized`.

## Traits and Traits Bounds
### Compilation and Dispatch

#### Static Dispatch

When you define generics in code, the compiler copy-pastes the code with the generic filled in for each type it's used for. This is called **static-dispatch**.

This process of going from generics to many non-generic types is called **monomorphization**. 

#### Dynamic Dispatch

The alternative is **dynamic dispatch**. It's a process where the address of the function to be called is looked up on runtime. This is done through the `dyn` keyword.

This is implemented through `vtables`, which are tables that store function pointers for an instance of a object.


This combination of a type that implements a trait and its `vtable` is called as a **trait object**.

To be implemented as a trait object or being **object safe**, a type must:
1. not have any methods that are generic or use the `Self` type.
2. not have any static methods.

> You'll prefer using static dispatch in your libraries and dynamic dispatch in your binaries.

### Generic Traits

The rust traits can be generic in 2 ways:
1. `trait Foo<T>` generic type parameter
2. `trait Foo {type Bar;}` associated type.

> Use an associated type if you expect only one implementation of the trait for a given type. Otherwise, use generic types.

### Coherence and Orphan Rule
**Coherence Property** - For any given type and method, there is only ever one correct choice for which implementation of the method to use for that type.

**Orphan Rule** - This says that you can implement a trait for a type only if the trait or the type is local to your crate.

**Blanket Implementation** - `impl<T> MyTrait for T where T: ...` - only the crate that defines a trait is allowed to write a blanked implementation for it. Adding a blanked implementation to an existing trait is considered a breaking change. 

**Fundamental Types** - Some types are so essential that it's necessary to allow anyone to implement traits on them. These are marked `#[fundamental]` and are `&, &mut, Box`.

**Covered Implementation** - These allow foriengn traits with forieng types. This only allowed when `impl<P1..=Pn> ForiegnTrait<T1..=Tn> for T0` is only allowed if at least one `Ti` is a local type and no `T` before the first such `Ti` is one of the generic types `P1..=Pn`.


### Trait Bounds

### Marker Traits

### Existential Types
