% Rust: Faster than C++, Safer than Java

Alexis Beingessner

Carleton University + Mozilla Research

[<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://cglab.ca/~abeinges)
[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://rust-lang.org)

rendered: http://cglab.ca/~abeinges/talks/ocics/

raw: http://cglab.ca/~abeinges/talks/ocics/index.md



# Rust!

A new systems programming language!

1.0 released this spring: http://rust-lang.org/

Great Free Books:

* https://doc.rust-lang.org/book/
* http://cglab.ca/~abeinges/blah/too-many-lists/book



# Why?

Web browsers are awful today

* Written in C++ for performance
* C++ is unsafe, complicated, and confusing
* Java/Go/Whatever non-starter because of GC



# Why?

Web browsers are awful tomorrow

* The free lunch from hardware is over
* Only place to grow is concurrency
* C++ is bad at concurrency



# Faster than C++

Technically impossible -- but true "at scale"

* More efficient defaults for many things
* Less implicit things, easier to see costs
* Don't need to copy/allocate for safety
* Easier to write big parallel code




# Faster than C++

Servo is a rewrite of Firefox's layout engine in Rust.

* Super fast, Massively parallel
* 2x faster than Firefox at layout with one thread
* 4x faster than Firefox at layout with four threads

https://github.com/servo/servo




# Safer than Java

Rust is completely memory/type-safe like Java:

* No dangling pointers
* No double-frees
* No buffer overflows
* No segfaults

All the usual stuff




# Safer than Java

But also has more:

* Tons more helpful static analysis
* No null pointers
* Statically impossible to invalidate views (iterator invalidation)
* Many concurrency footguns are gone

"If it compiles, it works"






# Practical and Usable

No big paradigm-shifting changes.

Still imperative and statically typed, like C++ and Java.

Has inference and other niceties that those languages lack.

Simple things are still simple (or simpler).




# Practical and Usable

Can seem harder to write at first; lots of compiler errors

"Fighting the compiler" is a common experience for new users

But most find that the compiler was actually right

(so this is just saving you time down the road)



# Practical and Usable

```rust
let mut bad_count = 0;
for name in names {
    if name.contains("cat") {
        println!("Kitty! {}", name);
    } else {
        bad_count += 1;
    }
}
println!("Found {} non-cats", bad_count);
```





# What does Rust do different?

Ownership!

Many problems boil down to bad data ownership




# Example 1

A classic mistake to make in C++:

```rust
fn to_string(input: &u32) -> &String {
    let string = format!("{}", input);
    return &string;
}
```

Trying to return a pointer to a local variable.

(this is not a problem with GC)





# Example 1

Doesn't compile in Rust:

```notrust
<anon>:3:4: 3:10 error: `string` does not live long enough
<anon>:3   return &string;
                  ^~~~~~
```

http://is.gd/LQdVoD


# Example 1

Not properly managing ownership.

Pointers *share* data, when really we wanted to give ownership
of the data to the caller:

```rust
fn to_string(input: &u32) -> String {
    return format!("{}", input);
}
```

GC is basically a rejection of true ownership



# Example 2

A classic mistake in *any* language:

```rust
let mut data = vec![1, 2, 3, 4, 5];
for x in &data {
    data.push(2 * x);
}
```

Trying to mutate some state while something else is viewing it (the iterator).

(This is a problem even with GC)





# Example 2

Doesn't compile in Rust:

```notrust
<anon>:4:9: 4:13 error: cannot borrow `data` as mutable because it is also borrowed as immutable
<anon>:4         data.push(2 * x);
                 ^~~~
```

http://is.gd/3Bet4V





# Example 2

An ownership issue: mutation needs uniqueness. (shared XOR mutable)

Functional avoids mutation always. (extreme)

> Aliasing with mutability in a sufficiently complex single-threaded program is effectively the same thing as accessing data shared across multiple threads without a lock





# What's Ownership?

Three core kinds of type:

* Values (`T`) - movable, mutable, destroyable, loanable
* Shared References (`&T`) - shared by many, immutable
* Mutable References (`&mut T`) - uniquely borrowed, mutable




# Values

Values can be used "at most once", meaning once they're consumed or moved,
they're gone forever.

Avoids using a value that you no longer own, or that was otherwise put in an
invalid state (freed its memory, closed its file, etc).




# References

Mutation and aliasing together are a mess, but individually useful.
Allow both but not at once; mutability "inherited" from access.

Useful in single-threaded, but generalizes to multi-threaded.

References are statically prevented from outliving their referent!




# Interior Mutability

Being able to mutate through shared references.

Relatively rare, but valid as long as ensures some level of mutual exclusion
(mutexes, atomics, and other thread-safe objects are obvious examples).

Single-threaded equivalents also exist for convenience. (we're pragmatic)





# That's It!

* Rust is faster than C++, safer than Java
* Still usable
* Accomplishes this through ownership

Any Questions?





