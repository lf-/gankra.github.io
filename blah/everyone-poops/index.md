% Pre-Pooping Your Pants With Rust

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">April 27, 2015 -- Rust Nightly 0.12.0</p>
</header>

> Wow I sure wrote this article, huh? What the fuck was I thinking, and why did everyone love this so much and latch onto it? I've preserved it for the sake of "history" but for real I made a much more useful version of this [in the Rustonomicon's section on leaking](https://doc.rust-lang.org/nightly/nomicon/leaking.html). Also I rebranded "Pre-Pooping Your Pants" as just "Leak Amplification".

# Leakpocalypse

Much existential anguish and ennui was recently triggered by [Rust Issue #24292: std::thread::JoinGuard (and scoped) are unsound because of reference cycles][24292]. If you feel like you're sufficiently familiar with Leakpocalypse 2k15, feel free to skip to the next section. If you've been thoroughly stalking all my online interactions, then you've basically seen everything in this post already. Feel free to close this tab and return to scanning my IRC logs.

The issue in question states:

> You can use a reference cycle to leak a JoinGuard and then the scoped thread can access freed memory

This is a very serious claim, since all the relevant APIs are marked as safe, and a use-after-free is something that should be *impossible* for safe code to perform.

The main focus is on the `thread::scoped` API which spawns a thread that can safely access the contents of another thread's stack frame *in a statically guaranteed way*. The basic idea idea is that `thread::scoped` returns a JoinGuard type (`jg` in `fn bad` in the above example).

JoinGuard's destructor blocks on the thread joining, and isn't allowed to outlive any of the things that were passed into `thread::scoped`. This enables really nice things like:

```
use std::vec::Vec;
use std::thread;
fn increment_elements(slice: &mut [u32]) {
    for a in slice.iter_mut() { *a += 1; }
}

fn main() {
    let mut v = Vec::new();
    for i in (0..100) { v.push(i); }

    let mut threads = Vec::new();
    for slice in v.chunks_mut(10) {
        threads.push(thread::scoped(move || {
            increment_elements(slice);
        }));
    }
    // when `threads` gets dropped here, `main` will block
    // on all the threads joining.
}
```

which dispatches some workload on every element in an array in 10-element chunks to separate threads (you'd probably want a pool for this; not the point). Magically Rust is able to statically guarantee this is safe without even knowing what a thread is! All it knows is that there's there's a Vec `threads` full of these JoinGuard things which borrow `v`, and thus can't outlive it. (Actually Rust also doesn't really understand Vec either. It *really* thinks `threads` is directly borrowing `v`, even if empty.)

Pretty nifty!

Unfortunately, **this is completely wrong**. This assumes that destructors are *guaranteed to run in safe code*. A lot of us in the Rust community (myself included) had grown to assume this was true. After all, we have a function that does nothing but drop an element without calling its destructor called `mem::forget`, and it's marked as `unsafe`!

Turns out this is simply a legacy detail from The Long Long Ago. There are in fact several ways to *write* mem::forget using only safe code provided by the standard library. A few of them can be regarded as simply implementation bugs, but one is fundamental: `rc::Rc`.

[Rc][rc] is our reference counted smart pointer type. It's actually pretty simple: you put data in a reference counted pointer with `Rc::new`. The reference count increases when you `clone` an Rc, and decreases when you `drop` one. No need to touch the reference count otherwise; the lifetime system will ensure that any internal references to the data are relinquished before the Rc they were obtained through is dropped.

On its own, Rc is totally fine. Because reference-counting is inherently a *sharing* of data, you can only get *shared* references to the internal data. This generally precludes *mutating* the internals of the Rc, so everything is frozen in time once it's put in there.

Except Rust has internal mutability. While sharing generally implies immutability in Rust, internal mutability is the exception. The [Cell][cell] types are the primary mechanism for internal mutability: they allow their contents to be mutated while being shared. For this reason they are marked as non-thread-safe. [`sync::Mutex`](mutex) is their thread-safe counterpart.

Now let's put those two tools together and write `mem::forget`:

```
fn safe_forget<T>(data: T) {
    use std::rc::Rc;
    use std::cell::RefCell;

    struct Leak<T> {
        cycle: RefCell<Option<Rc<Rc<Leak<T>>>>>,
        data: T,
    }

    let e = Rc::new(Leak {
        cycle: RefCell::new(None),
        data: data,
    });
    *e.cycle.borrow_mut() = Some(Rc::new(e.clone())); // Create a cycle
}
```

This is some janky nonsense, but long-story-short we can create reference-counted cycles using Rc and RefCell. The end result is that the destructor of the value put into our `Leak` type is *never* called, even though all of the Rcs have become unreachable! Never calling a destructor is actually fine on its own: you can abort the program or loop forever to similar effect. However in this case Rc *has told us that it has called the destructor*. We can verify this with this simple program:


```
fn main() {
    struct Foo<'a>(&'a mut i32);

    impl<'a> Drop for Foo<'a> {
        fn drop(&mut self) {
            *self.0 += 1;
        }
    }

    let mut data = 0;

    {
        let foo = Foo(&mut data);
        safe_forget(foo);
    }

    // super make sure there's no outstanding borrows
    // this will not compile if `foo` was not dropped!
    data += 1;

    println!("{:?}", data); // prints 1, should print 2
}
```

which [totally runs][safe_drop_example]. Now that we have `safe_drop`, let's get to the problematic in the issue that started this mess. Most of it is really just guaranteeing that the bug is reliably observable. It can be boiled down to the following:

```
fn main() {
    let mut v = Ok(4);
    if let &mut Ok(ref v) = v {
        let jg = thread::scoped(move || {
            println!("{}", v); // read on separate thread
        });
        safe_forget(jg); // now the thread won't be joined
    }
    *v = Err("foo"); // concurrent mutate with read; data race!
}
```

which also observes Undefined Behaviour, but less reliably.

This is unquestionably a flaw in *something* in the Rust standard library since it should be statically impossible to use-after-free or data-race in safe code, and we've managed to do *both*. The question is really *which* thing is wrong. The original issue placed the blame on `thread::scoped`, because it was posted by a compiler dev who was well aware that you could leak destructors in safe code.

However this triggered a huge amount of frustration in the community because everything they'd seen up until then had seemed to point to leaking destructors being unsafe. The `thread::scoped` API was *stabilized*. [Another API RFC][drain_range] was based on this assumption! `mem::forget` is marked as `unsafe` and that's all it does!

The only *real* problem seemed to be the alignment of the following stars:

* Reference Counting
* Internal Mutability
* Rc accepting data that doesn't live forever (non-`'static`)

If you remove any of the following then *there is no problem*. This of course resulted in people (again, myself included) variously demanding every possible combination of the above being removed. Yet more demanded an opt-in-built-in-trait `Leak` for specifying if your destructor can be leaked safely.

All of this would be *major* breakage and work with 1.0 *three weeks* away. So the status quo is almost certainly going to win: `mem::forget` is in fact totally safe, and code cannot assume this to be the case. `thread::scoped` must be redone.

Relevant RFCs are as follows:

* [Alter mem::forget to be safe][safe_forget_rfc]
* [Scoped threads, take 2][scoped_take_2]
* [Leak and Destructor Guarantees][leak_guarantees]

Note that Rust has never guaranteed *leaks* don't occur in general. Leaks, like race conditions (not to be mistaken with [data races][]), are such a vague concept that they're basically impossible to prevent. Putting stuff in a HashMap that you'll never ask for again is a kind of leak. Allocating stuff on the stack and then looping forever is a kind of leak. The case where you put it on the heap and forget to ever free it is a specific kind of leak that people seem particularly terrified of for whatever reason. The issue at hand is whether data that is statically known to be "gone" can fail to have its destructor called during normal, safe execution.

## Dealing With It

So initially I was pretty crestfallen. My work with collections had always assumed destructors were guaranteed to be run. This gave you the really nice property that you could make a special "handler" type that completely mangled the internal representation of a type *transiently*, and then fixed it when it was destructed. Paired with Rust's lifetime system, this guaranteed that the transient inconsistent state was *statically unobservable* (except by the handler, of course)! Rust code could be *more reckless* than C++!

The canonical example of this was to be `Vec::drain_range(a, b)`:

```
// do we need all these fields? Maybe? Don't want to think about it
// for an example!
struct DrainRange<'a, T> {
    vec: &'a mut Vec<T>,
    num_to_drain: usize,
    start_pos: usize,
    left: *mut T,
    right: *mut T,
}

impl<T> Vec<T> {
    // Produces an iterator that removes the elements in `self[a..b]`
    // Note: b is excluded per Rust range notation
    fn drain_range(&mut self, a: usize, b: usize) -> DrainRange<T> {
        assert!(a <= b, "invalid range");
        assert!(b <= self.len(), "index out of bounds");

        DrainRange {
            left: self.ptr().offset(a as isize),
            right: self.ptr().offset(b as isize),
            start_pos: a,
            num_to_drain: b - a,
            vec: self,
        }
    }
}

impl<'a, T> Drop for DrainRange<'a, T> {
    fn drop(&mut self) {
        // Drop all outstanding contents
        for _ in self { }

        let ptr = self.vec.ptr();
        let backshift_src = self.start_pos + self.num_to_drain;
        let backshift_dst = self.start_pos;

        let old_len = self.vec.len();
        let new_len = old_len - self.num_to_drain;
        let to_move = new_len - self.start_pos;


        unsafe {
            // Backshift all the elements!
            ptr::copy(
                ptr.offset(backshift_src as isize),
                ptr.offset(backshift_dst as isize),
                to_move,
            );

            // Tell the Vec to only consider this range of memory "used"
            self.vec.set_len(new_len);
        }
    }
}

// impl details for sake of completion
impl<'a, T> Iterator for DrainRange<'a, T> {
    type Item = T;

    fn next(&mut self) -> Option<T> {
        if self.left == self.right {
            None
        } else {
            unsafe {
                let result = Some(ptr::read(self.left));
                // let's ignore size_of<T> == 0 for simplicity
                self.left = self.left.offset(1);
                result
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for DrainRange<'a, T> {
    fn next_back(&mut self) -> Option<T> {
        if self.left == self.right {
            None
        } else {
            unsafe {
                // let's ignore size_of<T> == 0 for simplicity
                self.right = self.right.offset(-1);
                Some(ptr::read(self.right))
            }
        }
    }
}
```

This design is even unwinding-safe! Yay! ...but not destructor leak safe. Consider the following:

```
fn main() {
    let vec = vec![Box::new(1)];
    {
        let drainer = vec.drain_range(0, 1);
        // pull out and drop `box 1`, freeing the memory it points to
        drainer.next();
        safe_forget(drainer);
    }
    println!("{}", vec); // use after free of memory pointed to by `box 1`
}
```

This was troubling! I looked through some of the collections code in std to see if there was anything else that was relying on destructors for safety, and was pleased to see that there wasn't any that I could find (which isn't to say none exist)! I came to see that there was a vague hierarchy of how bad leaking a dtor is:

1. no dtor - leaking is irrelevant: primitives, pointers.
2. leaks system resources - not a *big* deal: Most collections and smaht pointer types, when they contain primitives will simply waste heap space if leaked.
3. leaks arbitrary other dtors - acceptable but nasty: Collections and smaht pointer types in general.
4. leaves the world in an *unexpected* but safe state - a scary footgun: `RingBuf::drain` should make the collection empty after the borrow its `Drain` iterator produces expires, but this is only guaranteed by the destructor as currently written. The collection *is* left in a consistent state otherwise.
5. leaves the world in a broken state - unacceptable: `Vec::drain_range` as written above.

Generally, one should strive to push their destructor reliance as far up this hierarchy as is practical, with the last category in the hierarchy *verboten* by an API that claims to be safe.

This hierarchy is based on the assumption that leaking a destructor should be considered a bug *in running code*. That is, application code should be welcome to assume that a leak doesn't randomly occur in code they control or under reasonable conditions in code they call. As is always the case in Rust, libraries have the onerous position of ensuring they are hardened against a leak.

The fact that an API *admits* causing a leak (such as Rc) should not be considered a bug in and of itself. Especially if the leak is convoluted to invoke (as is arguably the case with Rc). While `mem::forget` is going to be safe to call, it should generally still be regarded as something that *unsafe* code does.

## Pre-Pooping Your Pants

So how do we push `drain_range` up the hierarchy? It turns out this is totally trivial! Just add this single line to the `drain_range` constructor:

```
// Tell the Vec that it doesn't have any elements
// stored after the minimum bound.
unsafe { self.set_len(a); }
```

Now if we leak a DrainRange, we leak the destructors of the values that were supposed to be drained, as well as *completely lose* all the values that were supposed to be backshifted at the end. This is pretty squarely in the "unexpected but safe state" category, in addition to causing arbitrary other leaks. This is definitely *not great*, but still a *big* improvement over admitting accessing logically uninitialized memory!

This is what I call the Pre-Poop Your Pants (PPYP) pattern. Why Pre-Poop Your Pants? Well I like to think of the constructor-destructor pattern like a person eating. You eat some stuff (construction), process it, and then clean up any junk by pooping it out.

If a destructor gets lost, this is like *never pooping*. Dependent on the destructor, this can have varying consequences.

* Category 1 types are broken down completely by the body, and never needed to be pooped anyway.
* Category 2 types are basically benign, but clog up the system and may cause critical failure if constipation persists.
* Category 3 types are big bundles of stuff. Hopefully nothing in there is nasty!
* Category 4 types are like safe drugs. They're going to give your system a bad trip if they're not properly handled! If you're in a safe space when it happens you might be fine, but otherwise it can cause *really bad things*.
* Category 5 types are like dangerous drugs. You can overdose and die if they aren't properly handled, and should be carefully controlled.

The Pre-Poop Your Pants pattern is the following deviation from the normal digestion pattern: as soon as you consume something, be sure to poop the nastiest bits right into your pants. If everything goes according to plan, you can clean out your pants when you take your normal poop. If something goes wrong, you've got poopy pants but at least you're better off than otherwise.

The PPYP name emphasizes the following:

* It's a *suboptimal compromise*: no one wants poopy pants, but it's not the worst thing in the world.
* It's transient when things go right: If you poop your pants in the forest and no one hears, did it really happen?
* It's not always possible: the bad stuff may be too mixed in with the stuff that needs to be processed. The recipe may need to be completely rethought to avoid this (as is the case for `thread::scoped`).
* It's vaguely incoherent and meaningless on its own: this is a property inherited from its connection to the venerable Resource Acquisition Is Initialization (RAII) pattern.
* It's funny: I really just wanted to write about poopy pants for a while, and you actually just read it.

Oh, and [drain_range lives][]!


[24292]: https://github.com/rust-lang/rust/issues/24292
[cell]: https://doc.rust-lang.org/nightly/std/cell/
[rc]: https://doc.rust-lang.org/nightly/std/rc/
[mutex]: https://doc.rust-lang.org/nightly/std/sync/struct.Mutex.
[safe_drop_example]: http://is.gd/QSrQjK
[safe_forget_rfc]: https://github.com/rust-lang/rfcs/pull/1066
[scoped_take_2]: https://github.com/rust-lang/rfcs/pull/1084
[leak_guarantees]: https://github.com/rust-lang/rfcs/pull/1085
[drain_range]: https://github.com/rust-lang/rfcs/pull/574
[data races]: http://docs.oracle.com/cd/E19205-01/820-0619/geojs/index.html
[drain_range lives]: https://github.com/rust-lang/rust/pull/24781
