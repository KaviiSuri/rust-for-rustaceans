# Chapter 3 - Designing Interfaces

Your interfaces should be *[unsurprising](#Unsurprising), [flexible](#Flexible), [obvious](#Obvious) and [constrained](#Constrained)*. [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) are a good source to get guidance for this.

## Other Readings
- [Rust RFC 1105](https://rust-lang.github.io/rfcs/1105-api-evolution.html)
- [The Cargo Book on SemVer](https://doc.rust-lang.org/cargo/reference/semver.html)


## Unsurprising

User should be able to guess correctly about your interface as much as possible. Anything that can be *unsurprising* should be.

### Naming Practices

User infer a lot from the names. Thus, similar names should have similar functionality and different functionality should be named differently.

e.g. `.iter()` should take `&self` and return an `Iterator`
e.g. `.into_iter()` should take `self` and return an `Iterator`  

### Common Traits for Types

Somethings are expected to just work, e.g. `{:?}` or `Clone`. Eagerly implement most of the standard traits even if we don't need them immediately.

These traits are needed:
- `Debug` trait
- `Send` or `Sync`

If your type can't implement these, call it out in docs:
- `Clone`
- `Default`

Desirable traits: 
- `PartialEq`
- `PartialOrd`
- `Hash`
- `Eq`
- `Ord`

Add them with an opt-in feature(to serde for example):
- `Serialize`
- `Deserialize`


### Ergonomic Trait Implementations

Rust does not automatically implement traits for references to types that implement traits. You cannot generally call `fn foo<T: Trait>(t: T)` with a `&Bar`, even if `Bar: Trait`. This is because Trait may contain methods that take `&mut self` or `self`, which obviously cannot be called on `&Bar`.

Nonetheless, this behavior might be very surprising to a user who sees that Trait has only `&self` methods!

Thus, provide blanked implementations wherever appropriate for that trait for `&T where T: Trait`, `&mut T where T: Trait`, and `Box<T> where T: Trait`

Iterators are another case where you’ll often want to specifically add trait implementations on references to a type.


### Wrapper Types
If you provide a relatively transparent wrapper type (like Arc), there’s a good chance you’ll want to implement `Deref` so that users can call methods on the inner type by just using the `.` operator.


If accessing the inner type does not require any complex or potentially slow logic, you should also consider implementing `AsRef`

`From<InnerType>` and `Into<InnerType>` wherever possible.



Borrow is intended only for when your type is essentially equivalent to another type, whereas `Deref` and `AsRef` are intended to be implemented more widely for anything your type can “act as.”

## Flexible
Avoid imposing unnecessary restrictions and to only make promises you can keep.
- Restrictions come from trait bounds and argument types.
- Promises come from Trait implementations and return types.

Each make different restrictions and promises:
```rust
fn frobnicate1(s: String) -> String
fn frobnicate2(s: &str) -> Cow<'_, str>
fn frobnicate3(s: impl AsRef<str>) -> impl AsRef<str>
```

### Generics vs Dynamic Dispatch

One way to make your interface flexible is to use Generic Arguments. This has pros and cons. While it does improve the ergonomics without runtime cost, it does entail larger binary sizes and also makes the code less readable and tough to write.

The other way to handle this would be using `dynamic dispatch`. This has runtime costs but (usually) they are negligible. There are 2 problems here:

1. You make this choice for your user (who might not want dynamic dispatch).
2. dynamic dispatch or `impl Traits` are manageable for simple bounds, but for complex trait bounds, rust doesn't know know how to construct the `vtable`, so you can't have `&dyn Hash + Eq` as a bound.
3. With generics, the caller themselves can choose `dynamic dispatch`by passing in a trait object. The reverse is not true.

Starting off with concrete types and then turn them generic over time can break backward compatibility.

### Object Safety

When you define new trait, it's object-safety is an unwritten part of the trait's contract. If it is, users can treat different types that implement it as a single common type using `dyn Trait`. Thus, **prefer your traits to be object-safe even if that comes at a slight cost to ergonomics of using them**.

If your traits needs generic methods, try adding the generic type parameters into the trait itself or using dynamic dispatch to preserve object safety of the trait.

Alternatively, adding a `where Self: Sized`  trait bound to that method can make it possible to call the method only with a concrete instance of the trait (not through a `dyn Trait`).

Object safety is  a part of you public API!

### Borrowed vs Owned

Generally, this is obvious when you implement the logic. If you need the data to be owned, better make the caller just pass owned data instead of cloning it.

This decision however, gets complicated if you have an interface with lifetimes so complex that they  become hard to use.

### Fallible and Blocking Destructors

I/O based types often need to perform cleanup if they're dropped. This cleanup itself may entail I/O calls. The obvious place to do this is the `Drop` trait implementation. However, if the cleanup fails, there is no way to communicate that error as the value has been dropped.

There is no clear cut solution to this, but one thing we can do to facilitate the users who don't want to leave loose threads is provide a explicit destructor method with `-> Result<_, _>` output or asynchrony `using async`. 

There are a few problems with this too! The `Drop` trait needs a `&mut self`, which means that it'll still be called and you can't just use the destructor and ignore result.

A solution to this might be making the top level type an `Option` and handle both cases. You can do `Option::take` in both destructors and call the inner explicit destructor only if the inner type is not `None`.

A second workaround to this is to make each of your fields `takebale`.

The 3rd option is to hold the data inside the `ManuallyDrop` type, which dereferences to the inner type. Then, there will be no unwraps. You can also use `ManuallyDrop::take` in drop to take ownership at destruction time. The primary issue with this approach is that `ManuallyDrop::take` is unsafe.

## Obvious

It's critical for you to make it as easy as possible for the users to understand your interface and as hard as possible for them to use it incorrectly.

The 2 primary tools at your hand for this is the Type System and Documentation.

### Documentation

- Clearly document any cases where your code may do something unexpected or relies on the user doing something beyond what's implied by the type system. Panics are a good example of both of these.
- Include end-to-end usage examples for your code on a crate and module level.
- Organize your docs. Use modules to group together semantically related items. And use links to interlink your docs.
- Enrich the docs, add external links to sources that explain the concepts, data structures, algorithms and other aspects.


### Type System Guidance

- **Semantic Typing** - Add types to express the meaning of a value, not just its primitive type. e.g. preferring enums over boolean
- **zero-sized types to indicate that a particular fact is true about an instance of a type**
- **#[must_use]** - is an annotation that warns the user if they ignore the return value.

## Constrained

Overtime, the user will depend on every property of your interface, weather it's a bug or a feature.

The following are backward incompatible changes that can be missed:


### Type Modifications
Removing or renaming a public type will break someones code. Use `pub(crate)` and `pub(in path)` wherever possible to avoid this. The less public types you have, the more freedom you have to change things.

Adding a private field to a publically exposed struct can also cause problems. The user might instantiate the type directly with the `StructName {..}` syntax which will break. Rust provides `#[non_exhaustive]` attribute that can prevent the use of implicit constructors and non-exhastive pattern matches on the type.


### Trait Implementations

Adding a blanket implementation of an existing trait is a breaking change since the user might already be implementing the trait for some type. 

Same holds true for implementing foreign traits for existing type or existing trait for foreign type.

Removing a trait implementation is a breaking change, but implementing traits for new type is never a problem.

Most changes to existing traits are also breaking changes, though adding a new method with default implementation can be fine. We do have `sealed traits` which are traits that can only be used, and not implemented. They are most commonly  used for derived traits.

### Hidden Contracts

#### Re-Exports

If any part of your interface exposes foreign types, then change to one of those types is also change to the interface. To mitigate this, it's generally better to wrap foreign types using the newtype pattern and only expose parts of the type that are useful. Most of the time, this can be avoided by using `impl Trait`.


### Auto-Traits

Rust has some traits that are implemented automatically e.g. `Send`, `Sync`, `Unpin`, `Sized`, `UnwindSafe` etc. These are hidden promises that are made to the user, which might break in the future.
