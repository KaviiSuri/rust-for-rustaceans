# Chapter 1: Foundations

## Memory

### Value
Combination of type and an element of that type's domain of values. A value can be turned into a sequence of bytes using that type's *representation*.

e.g. `"Hello World!", 6`

### Variable 
A named slot on the stack, a *place* to store a value. 


#### High-level
On a high-level, we think of variables as names given to values.
When you assign a variable, the value has been given a name. When you use it, you can imagine a *dependency  line*  or *flows* going from the previous to next access. If a variable is moved, no more lines can be made.

Each *flow* traces lifetime of a particular instance of a value. They can fork, merge and split too. The compiler can thus make sure that all parallel flows are compatible with each other.

Thus a variable exists as long as it has a legal-value (it's initialized and not moved).


#### Low-Level
A variable is a value slot. When you assign to it, the slot is filled, and the old value is dropped. A pointer to a variable refers to the variable's backing memory and can be dereferenced to get its value.


### Pointer 
A value that holds address of a region of memory. A pointer points to a place. 


### Memory Regions

#### The stack
Stack is segment of memory that your program uses as a scratch space of function calls. A *frame* is allocated on each function call containing all the local variables and arguments. When the function return, the space is reclaimed

```md
  Intial
 --------
 | main |
 --------

 `fun` function called

 --------
 | fun  |
 --------
 | main |
 --------


 `fun` function returned
 --------
 | main |
 --------
```
> Any variable stored in a frame cannot be accessed when that frames goes away, so any reference to it must have a lifetime <= lifetime of frame


#### The Heap
The heap is a pool of memory that isn't tied to current call stack of the program. Values here live until they are deallocated.


In rust, you interact with the heap using the `Box` type. When you do `Box::new(value)`, the value is stored on the heap, and what you get back is a pointer to that value. When the `Box` is dropped, that memory is freed automatically. This prevents from the memory from leaking.

But if you actually want to leak memory, you can use `Box::leak`, which get's you a `'static` reference.


#### Static Memory
Values here live for the entire execution of your program. It contains your program code (read-only), variables declared static and certain constants.

This is where the name for `'static` comes from, although they do not necessarily point to static memory.

### Ownership
**All values have a single owner.** i.e. Exactly one location is responsible for ultimately deallocating each value. This is enforce by the borrow checker. If value is moved (e.g. by assigning a new variable, pushing in vector etc), you can no longer access the value through the variable that flow from the original variable.

The `Copy` trait let's variable by-pass this by actually copying data instead of just moving it.

When a value's owner no longer has use for it, it's responsibility of the owner to do necessary cleanup before dropping it. Generally, this is done recursively.


### Borrowing and Lifetimes

**References** are pointers that come with additional constraints on how they are used and what access do they have.


#### Shared Reference `&T`
It is a pointer that may be shared. Any number of other references can exist to the same value. Each reference implements `Copy`. The compiler assumes that the value shared reference points to *will not change* while the reference lives.


#### Mutable References `&mut T`
These are exclusive references, no other mutable or shared reference may exist for it's lifetime.

#### Interior Mutability
Some types provide this property, i.e. they allow you to mutate a value through a shared reference.

2 Types:
1. **Allows you to get mutable reference from a shared reference:**: `Mutex`, `RefCell`, they have certain safety measures to ensure that, for any value, they give a mutable reference to only once at a time. Under the hood, they use `UnsafeCell`.
2. **Allows you to replace a value given only shared reference:** `std::sync::atomic` and `std::cell::Cell`.

#### Lifetimes
It is a name for a region of code that some reference must be valid for. 

##### Borrow Checker
Whenever a reference with some lifetime `'a` is used, the borrow checker checks that `'a` is still alive by going back to where the reference was taken from and checking for conflicting uses along it.

##### Generic Lifetimes
You might need to store references inside types, in that case, you will need to use generic lifetimes so that the borrow checker can check their validity when they are used in the various methods of the type.

Caveats:
1. If your type implements `Drop`, then it'll not be dropped before making sure that all the lifetimes are legal while dropping.
2. Although you can multiple generic lifetimes, it mostly just complicates things. Use multiple generic lifetime parameters if you have a type that contains multiple references, and its methods return references that should be tied to the lifetime of only one of those references.


##### Lifetime Variance
If `'b`: `'a` (`'b` outlives `'a`), then `'b` is a subtype of `'a`. A type can be
1. **Covariant** - if you can just use a subtype in place of the type.
2. **Invariant** - you must provide exactly the given type.
3. **Contravariant** - Comes up for function arguments. Function types are more useful if they’re okay with their arguments being less useful. `Fn(T)` is Contravariant in `T`.

