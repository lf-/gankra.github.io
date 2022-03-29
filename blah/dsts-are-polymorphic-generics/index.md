# DSTs Are Just Polymorphically Compiled Generics

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">March 30th, 2022</p>
</header>

> This post was written on behalf of eddyb, to help them express a design detail of Rust that they understand to be fundamentally true, but isn't properly described/documented anywhere. Hope it helps!

Rust has a feature called DSTs ("dynamically-sized types"), which allows you to have a pointer to some data with a size that's unknown at compile time. This is basically used everywhere with "slices" (`&mut [T]`, `&str`) and a critical component of "Trait Objects" (`Box<dyn MyTrait>`).

A DST pointer is "wide" because it must hold both the normal pointer you expect *and* some dynamic *Metadata* that makes it possible to handle the DST. There are currently 3 kinds of metadata:

* [Thin](https://doc.rust-lang.org/core/ptr/traitalias.Thin.html) = (), no Metadata needed.
* A slice's length = `usize`
* A trait object's vtable pointer = `&'static VTable`

The two non-trivial Metadatas come from the two Fundamental DSTs that are built into the language itself: `[T]` and `dyn Trait` ([extern type](https://rust-lang.github.io/rfcs/1861-extern-types.html) is [Thin](https://doc.rust-lang.org/core/ptr/traitalias.Thin.html)). Users of Rust can create new DSTs in one of two ways:

* Newtyping a DST and transmuting
* Creating a generic type and [Unsizing It](https://doc.rust-lang.org/reference/type-coercions.html#unsized-coercions).

For an example of the former, we can look at `str` which is just `struct str([u8])` and is constructed by creating a proper slice pointer and transmuting it (`&[u8]` => `&str`).

For an example of the latter, you could define something like:

```rust
struct MyGeneric<T> {
    some_field: bool,
    data: T,
}
```

And [Unsize](https://doc.rust-lang.org/reference/type-coercions.html#unsized-coercions) `&MyGeneric<[T; N]>` to `&MyGeneric<[T]>`. 

In either case, we have introduced some level of "distance" from the Fundamental DST by introducing extra structs that wrap them. These wrapping structs are forced to become DSTs but *crucially* the Metadata is unchanged: Rust still only stores the Metadata for the Fundamental DST and can figure everything else out from there.





# Very Large Dynamically Sized Type Metadata (VLDSTM)

Currently, all DSTs must follow these rules:

* **Indirection**: A DST must be "completed" by being put behind a pointer (`&`, `*mut`, `Box`, ...)
* **Solitary**: A pointer can only point to one Fundamental DST (only one Metadata)
* **Trailing**: The Fundamental DST must be "trailing" (field offsets cannot depend on Metadata)

This post isn't touching the first condition, sorry by-value DST lovers. We will however be exploring what it means to loosen the other conditions.

The Trailing rule largely exists for simplicity, and can seemingly "simply" be removed with sufficient work on the compiler to support it, except for one case: *generics*. Generics let us name a type and repeat it multiple times in a struct. The simplest example of this is arrays: `&[dyn MyTrait; 8]` takes "one" DST and turns it into "eight". What does this mean? Does this require more Metadata?

This brings us to the Solitary rule. There are two ways to break the Solitary condition: by having multiple Fundamental DSTs that are *neighbouring* or *nesting*.

A type like `(dyn MyTrait1, dyn MyTrait2)` has *neighbouring* Fundamental DSTs.

A type like `[dyn Trait]` has *nested* Fundamental DSTs.

In either case, we clearly need *multiple* Metadata, making our pointer wider and wider. We're talking *SIMD* levels of wide. Actually no we're talking *Itanium* levels of wide with VLDSTM pointers (Very Large Dynamically Sized Type Metadata)!

But how many Metadatas do these types actually contain? Well when looking at a type like `&[&dyn Trait]` we have *arbitrarily many* Metadata, because each nested `&dyn Trait` can be a *different* implementer of Trait. Does that mean `&[dyn Trait]` must be Infinitely Wide? As it turns out, no! The reason there are arbitrarily many Metadata is because there are arbitrarily many *pointers*. Each pointer gets its own independent Metadata. With a type like `&[dyn Trait]` there is only one pointer, so *everyone must share*.





# Polymorphic Compilation Of Generics

To understand this, let's think about a simple generic function:


```rust ,ignore
fn my_generic<T: Clone>(val: T) {
    let a = val.clone();
    let b = val.clone();
}
```

This code can handle *any* type T that implements Clone -- but hold on, we're passing it around by value! Doesn't the Indirection rule tell use that Rust thinks it's "impossible" to do this? Indeed, it does, and that's why Rust cheats: it *monomorphizes* the generics away, which is just a fancy way of saying that whenever you call this function, Rust generates a copy-pasted version with all T's replaced with the actual type you're using. So if you pass it a u32, rust will just make:

```rust ,ignore
fn my_generic_u32(val: u32) {
    let a = val.clone();
    let b = val.clone();
}
```

Which of course Rust can happily deal with. Rust applies the exact same strategy for generic *types* too -- it's copy-pasting all the way down. This is a Simple But Effective approach, but it has one downside: you can't have a generic function pointer!

Rust will happily let you turn `my_generic_u32` into `fn(u32) -> ()`, but won't let you make `my_generic` into `fn<T: Clone>(T) -> ()`. All monomorphization is handled *statically* (at compile time) and creates a *new* function pointer for each type substitution. Even if we got around the whole "multiple function pointers" thing with a vtable, it still wouldn't be good enough because a function pointer is a *dynamic* (runtime) construct, and so the compiler *can't* predict all the monomorphizations. 

Languages like Java solve this problem by just making all types Indirected all the time. And that's why languages like Rust with inline layouts can't have polymorphic generics.

Oh I didn't see you there Swift! Isn't this problem annoying for us inline-layout languages? Sorry what? [You have polymorphic generics](https://gankra.github.io/blah/swift-abi/#polymorphic-generics)??? HOW??? I *JUST* finished explaining how that's impossible!

Swift *does* need to Indirect polymorphic stack variables with boxing, but once you get past the stack "roots" all the actual values have the same layouts they would have with monomorphization! Swift accomplishes this with something it calls *Value Witness Tables*. Value Witness Tables are just vtables full of information about the type: size, align, stride (spicy size), its clone impl, its move impl, etc.

Whenever you have a generic function like

```swift
func SwiftyGeneric<T, U, V>(arg1: T, arg2: U)
```

And you ask Swift to turn that into a function pointer, what it *generates* is something like this:

```swift
func SwiftyGeneric<T, U, V>(
    arg1: Pointer<T>, 
    arg2: Pointer<U>, 
    witness_T: ValueWitnessTable, 
    witness_U: ValueWitnessTable,
    witness_V: ValueWitnessTable
)
```

Now inside the *body* of SwiftyGeneric, any time we need to handle an instance of our generic types, we can just ask the Value Witness Table whatever we need to know, and we can even *forward* it to other generic code. Need to make an `Array<T>` inside SwiftyGeneric? No problem, just hand the Array's code your `witness_T` and it can use `witness_T.size/align/stride` to figure out how much memory to allocate and all the offsets!

I'm making it sound simple but I cannot emphasize enough how much work this is to actually do. In particular, the fact that there can be generic types instantiated inside your generic function pointer means *you need to be able to generate Value Witness Tables at runtime* (and because Swift has stuff around unique class ids, there needs to be a global type-keyed registry of these things 🙀). Rust *tried* to have polymorphic generics in the early pre-1.0 days, and they quite reasonably *gave up* because it was too much work. For real Swift, great fucking working for getting all of this to work!





# Metadata Are Value Witnesses

Huh, I went off on a bit of a tangent there, huh? No of course not! Read the title of this section! *Metadata are just Value Witnesses*. The Metadata for `dyn Trait` is *literally* just a Value Witness Table with `Trait`'s methods stapled to the end! Slices don't need that much, so we just need a Value Witness Length (usize).

Crucially, these polymorphic generics are *WAY* tamer than full-on polymorphic function pointers. The whole "generating Value Witness Tables at runtime" thing goes completely away and you can indeed generate all your Metadata (Value Witnesses) ahead of time, which is... A Lot More Tractable!

Now with that established, we can return to our hairy questions about Multiple Metadata. What *does* `&[dyn Trait]` *mean*? It means this:

```rust ,ignore
&<T: Trait, const N: usize>[T; N]
```

It's just polymorphic generics! DSTs are just polymorphic generics! Although we aren't "allowed" to name a type variable, so it's *actually* more like:

```rust ,ignore
&[impl Trait; impl const usize]
```

But that's "just" sugar for the first version.

With our DSTs desugarred to generics, all of our answers become pretty simple:

```rust ,ignore
// Everything is generics, so nesting just makes all copies the same:

syntax:     &[dyn Trait],
interpret:  (&[T; N], (N = usize, T = &VTable)),
stripped:   (&void, (usize, &VTable)),
```

```rust ,ignore
// The problematic case of non-trailing is just a special case of nesting:

syntax:     &[dyn Trait; 8],
interpret:  (&[T, 8], (T = &VTable)),
stripped:   (&void, (&Vtable)),
```


```rust ,ignore
// Neighbouring is just having a fresh type variable for each Fundamental DST:

syntax:     &(dyn Trait, [u8], u32, dyn Trait),
interpret:  (&(T, [u8; N], u32, U), (T = &VTable, N = usize, U = &VTable)),
stripped:   (&void, (&VTable, usize, &VTable)),
```

And that's it! That's what it would *mean* for Rust to loosen its restrictions and enter The World Of VLDSTM: An extra Metadata for each new generic, and having to dynamically compute the offsets of arbitrary fields using the Metadata, instead of just relying on the Trailing Rule to handle anything more complex than array stride.
