% Rust, Ownership, and Iteration

Alexis Beingessner

Carleton University + Mozilla Research

[<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://cglab.ca/~abeinges)
[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://rust-lang.org)

rendered: http://cglab.ca/~abeinges/talks/iter-easy/

raw: http://cglab.ca/~abeinges/talks/iter-easy/index.md



# Rust!

A new systems programming language!

1.0 released this spring: http://rust-lang.org/

Great Free Books:

* https://doc.rust-lang.org/book/
* http://www.oreilly.com/programming/free/files/why-rust.pdf
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




# What's Rust?

* Safer than Java!
* Faster than C++!
* No Garbage Collection!
* Practical and Usable!
* Fearless Concurrency!

http://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html



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





# Getting Comfortable With Ownership

Now that we have a vague idea of what Rust is all about,
let's look at a concrete and practical problem.

Any Questions?





# Let's talk about this for 30 minutes

```rust
pub trait Iterator { // I can yield things
    type Item;       // Things I yield
    // How I yield them
    fn next(&mut self) -> Option<Self::Item>;
}
```

* `mut` because we want to *progress*
* `Option` coalesces `has_next` and `get_next`
  * Some(item) - I have next, and here it is
  * None - I don't have next





# Usage

```rust
for i in 0..10 {
    println!("{}", i);
}
```

* `i`: name for `item` in `Some(item)`
* `0..10`: the thing to iterate (0 to 10, 10 excluded)






# Collection Iterator Dream Team

* IntoIter
* Iter
* IterMut

(these are structs)





# OMG SO MANY WHY??

Ownership!





# IntoIter - Owned Values (T)

* Moves the data out of the collection
* You get total ownership!
* Can do anything with data, including destroy it





# What if we only had IntoIter

```rust
fn process(data: Vec<String>) {
    for string in data.into_iter() {
        // I got all the data, it's mine!
        println!("{}", string);
    }

    // Oh no! Iterating consumed it :(
    println!("{}", data.len()); //~ERROR
}
```

Trash Language; Quit Forever





# Iter - Shared References (&T)

* Shares the data in the collection
* Read-only access
* Can have many readers at once





# Iter lets you look but not touch

```rust
fn print(data: &Vec<String>)
    for string in data.iter() {
        // All I can do is read :/
        println!("{}", string);
    }

    // Yay it lives!
    println!("{}", data.len());
}
```





# Everyone can look at the same time!

```rust
fn print_combos(data: &Vec<String>) {
    let len = data.len();
    for string in data.iter() {
        // Iterate and query at the same time!
        println!("{} {}",
                 string,
                 &data[random_idx(len)]);
    }
}
```



# IterMut - Mutable References (&mut T)

* Loans the data in the collection
* Read-Write access
* Only one loan at once




# IterMut gives you exclusive access

```rust
fn make_better(data: &mut Vec<String>) {
    for string in data.iter_mut() {
        // Ooh I can mutate you!
        string.push_str("!!!!!");
        // But I can't share :(
    }
}
```




# Drain - I Drink Your Milkshake (🍼T)

* Partially moves the data
* Full access to the elements
* Doesn't destroy the container





# Drain lets you partial move

```rust
let mut data = vec![0, 1, 2, 3, 4, 5];

for x in data.drain(2..4) {
    // got exclusive access so we can "drain"
    // the values out but leave the vec alive
    consume(x);
}

// data lives on! We can reuse the allocation!
// Bulk `remove`!
// Dat Perf. ;_;
assert_eq!(&*data, &[0, 1, 4, 5]);
```





# Recap

Iterators naturally fall out of ownership:

* IntoIter - `T`
* Iter - `&T`
* IterMut - `&mut T`
* Drain - `🍼T`






# IterMut is kinda weird...

Rust doesn't understand when indexing is disjoint:

```
let mut data = vec![1, 2, 3, 4];

let ptr1 = &mut data[0];
let ptr2 = &mut data[1]; //~ ERROR

*ptr1 += 5;
*ptr2 *= 3;
```

(I would argue this is a good thing)




# IterMut is kinda weird...

But it's ok with IterMut?!
Ownership busted?!

```
let mut data = vec![1, 2, 3, 4];
let mut iter = data.iter_mut();

let ptr1 = iter.next().unwrap();
let ptr2 = iter.next().unwrap();

*ptr1 += 5;
*ptr2 *= 3;
```





# IterMut 100% Legit and Safe

* Iterators are "one shot"
* Each element yielded *at most* once
* Can't get a fresh IterMut while refs live

(not true for indexing in general)





# Why does the API allow this?

```rust
impl<'a, T> Iterator for IterMut<'a, T> {
  type Item = &'a mut T;
  fn next<'b>(&'b mut self) -> Option<&'a mut T> {
    ...
  }
}
```

`'a` is not associated with `'b`, so `next` doesn't care if you still have a `'a`

*Shh... It's okay borrow checker, only dreams now*






# Crazy!

How can Rust let you implement this?!?






# The Borrow Checker is Crazy Smart

You can statically prove to the compiler that this works.





# IterMut for an Array

```rust
# use std::mem::replace;
pub struct IterMut<'a, T: 'a> { data: &'a mut[T] };

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        let d = replace(&mut self.data, &mut []);
        if d.is_empty() { return None; }

        let (l, r) = d.split_at_mut(1);
        self.data = r;
        l.get_mut(0)
    }
}
```


# Not Just Arrays

Same basic idea works on singly-linked lists and trees -- anything with clear
ownership that you can get subviews into.





# Conclusions

* Rust is safer than Java, faster than C++, still usable
* Ownership!
* Iter, IterMut, IntoIter, and Drain "natural" reflections of ownership
* Iterator always lets you call `next` again (consequences)





