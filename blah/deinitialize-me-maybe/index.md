% Destroy All Values: Designing Deinitialization in Programming Languages

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">January 23rd, 2022</p>
</header>

> Disclaimer: this was entirely written in a fever dream over two days, I have no idea what this is.

So you're making a programming language. It's a revolutionary concept: the first
language optimized for the big bappy paws of cats. It's going great -- you've got yourself a sophisticated LR(BAPPY) parser; some basic types like Int, and FoodBowl; and some basic operations like `if`, and `purr`.

But now you've reached that dreaded point: you have to implement some kind of heap allocation. Well that part's fine. The real nasty part is *deallocation*.

It's time. It's time to destroy some values.

And so you build a world-class concurrent tracing generational garbage collector and call it a day. 

Problem Solved.

But is it really? When we talk and think about "destructors" or "deinitialization" the primary focus is always on memory management, and for a good reason: it's incredibly pervasive, involves Spooky Action At A Distance, and if you mess up then the way everything breaks is *horrible*.

But in writing a big piece of complex software, you're invariably going to have to things that aren't valid yet, or aren't valid anymore. The classic example that's Basically Memory But Not is *files*. When accessing a file, it will go through several phases:

1. The file isn't opened yet (uninitialized)
2. You have requested the file to be opened (*maybe* initialized)
3. You have checked the success status (initialized)
4. You close the file (deinitialization)
5. The file has been closed (uninitialized)

These states logically happen, and to varying extents they are states a programmer must think about. This is true regardless of if you're making a super dynamic scripting language where you can mutate the parser at runtime (just remove the file handle's variable from the AST, duh!) or some towering statically compiled monument to the power of type systems (just encode the file handle with affine higher-rank dependent modules, duh!).

And like, you don't even need to talk about "resources" like memory or files. It's pretty natural to write a program where state changes, and things conditionally exist. You need a way to talk about whether something has an event handler that needs to be run, or what to do when a map doesn't contain the key you're looking for.

These are things the programmer has to think about. Usually these things are expressed with sentinel values like "null" or "None", which are basically just "there is no value" (i.e. basically the same thing as uninitialized memory, but hoisted up a logical level and less chaotic). 

Even if your language has no notion of things being initialized or uninitialized, it's still happening on some semantic level, and it's happening *constantly*. So all programming languages generally develop some tools to help programmers describe and think about those situations.

Let's look at those tools, and the problems that try to solve (and introduce!).





# A Brief Aside: Language Design is Holistic

The nasty thing with programming languages is that good design is often a *holistic* property. Certain features only make sense in the presence (or absence!) of other features. We're going to be exploring a lot of different concepts in this post, and it's important to keep in mind that they don't work in a vacuum. They solve specific problems caused by solutions to *other* specific problems, caused by solutions to **other other** problems.

For instance: 

"I don't want to have `null` to be a valid value for every type"

* Then you really want something like [an `Option<T>` type to express the cases where something is missing](https://doc.rust-lang.org/rust-by-example/std/option.html).

* Well since we have `Option<T>` [why not have `Result<T, E>` for error handling](https://doc.rust-lang.org/rust-by-example/std/result.html).

* Now since we have all this machinery for tagged unions, we might as well let users [define their own (enums that can contain values)](https://doc.rust-lang.org/rust-by-example/custom_types/enum.html).

* Wow now that we have tagged unions everywhere we're really hurting for [pattern matching, better add that too](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring/destructure_enum.html)!

* Wow a full on match (switch) statement is overkill for lots of cases, let's [add `if let` to make it easier to do a fallible pattern match](https://doc.rust-lang.org/rust-by-example/flow_control/if_let.html).

* Well if-let is really awkward if it's trying to guard against a specific case, let's [add `guard let` to make it easier to do early returns when patterns don't match](https://docs.swift.org/swift-book/LanguageGuide/ControlFlow.html#ID525).

* Hmm lets also add closures to the language so you can do `map(some_option, |x| x+1)`

* And chainable methods so you can write out the maps in a more intuitive order!

* etc, etc, etc

-------

All of these features together make a nice and coherent system that a lot of people swear by.

But if you have a language where `null` is already a pervasive thing, the value of adding even just *Option* is a lot more dubious for two reasons: 

* You can end up writing stuff like `Option<NotNull<T>>` everywhere to try to describe the same thing as `T` (but more explicit about intent). And there's extra friction everywhere because all the legacy code wasn't built around this system!

* To make Option "feel good" you need to bring all of its buddies like Result, guard-let, closures, method chaining, etc. This is a huge undertaking (and not immediately obvious to do), so usually the first few versions of the feature will be extra miserable to use.

Quite understandably, a lot of people are going to see that design and go "wow Options are garbage". Meanwhile folks using languages built from the ground up without Option swear by how great it is to use! This is because one language has a more coherent and holistic design, while the other has a more jumbled and stapled together one.

You might be picturing some particular languages here, but like, I don't think the languages this happens to are *bad* or *wrong*. They're just old and actively developed, and have had time for this stuff to build up. I believe this to be the fate of all actively developed languages with backwards compatibility guarantees: slowly grow into a horrible shambling mass as each new feature becomes increasingly awkward to bolt onto the side.

(Arguably Rust and Swift both going through a relatively long period of aggressive breakage before stabilizing was an enormous boon, because wow they both tried a lot of Stuff That Did Not Work, and they would be a lot messier languages if they couldn't throw stuff out and rework everything around the parts that did. But worry not, one day they will blossom into shambling frankensteins of their own making.)





# Background Concepts

You can skip this section if you already believe you understand these concepts.

TLDR:

* Background Concept 1: We're mostly concerned with *uninitialized memory*, which describes whether we can reasonably make sense of the value stored in that memory. It helps us write correct programs (seriously, it's good!), and optimize them. *Sentinel values* (i.e. null) are a form of *logical* uninitialized memory that the programmer can manipulate at runtime. 

* Background Concept 2: Establishing terminology for copy/move semantics. Examples will be done in a "pseudo-Rust" language called "ownershiplang". Notable operations:
  * `let x = MyType()`: call MyType's constructor and init a new var x with it
  * `relet x = MyType()`: call MyType's constructor and reinit an existing var x with it
  * `drop x`: run destructor of x and deinitialize the variable
  * `copy x`: bitwise (shallow) copy x
  * `move x`: bitwise (shallow) copy x, and deinitialize the variable
  * `clone x`: deep copy x 
    * What C++ calls a "copy constructor"
  * `y.clone_from(&x)`: update `y` to be a clone of `x` without dropping `y` 
    * What C++ calls a "copy-assign constructor/operator"
  * You could define move_from to express C++ move-ctors but it would be messy and weird.

* Background Concept 3: Definite Initialization analysis lets you know how initialized variables are at each point in the program. This article will frequently say "if you use Definite Initialization then...", so it's useful to understand the concept.



## Background Concept 1: What Is Uninitialized Memory, And Why It's Useful

The [accidental prequel to this post](https://gankra.github.io/blah/initialize-me-maybe/) has a lot of discussion on the concept of "uninitialized", and how it can be more murky then you'd like. 

For the purposes of this discussion, we are interested in a fairly high-level concept of initialization: whether a variable (or other location in memory) contains a useful value.

Usually when talking about things being uninitialized, compiler people are talking about *uninitialized memory*. That is, when a variable comes into scope it gets allocated a little plot of memory that contains *nothing*. The bits are not 0 or 1, they are a third value: `uninit`.

What this is trying to express is that the memory contains irrelevant garbage that shouldn't be looked at. Only once we *initialize* the variable by assigning a value to it (overwriting the random garbage bits) can we confidently start reading that memory and know that it will have a useful meaning.

It's really important to understand this part: **uninitialized memory is purely a logical construct. It exists to help us reason about the semantics of a program. We can arbitrarily decide that some particular memory is or isn't uninitialized if we find it useful.**



### Uninitialized Memory Is Good Actually

So why *is* it useful to talk about uninitialized memory? For compiler developers, there are two major reasons:

1. Checking: We want our programs to do useful things, and uninitialized memory is *by definition* useless. By building systems that can identify when memory is logically uninitialized, we can build systems that help the programmer avoid using it, which helps them make useful programs.

2. Optimization: Because uninitialized memory is by definition useless, the compiler doesn't need to care about preserving it, and can potentially discard anything that *tries* to use it. That is, most uses of uninitialized memory are blatantly Undefined Behaviour.

So if the compiler sees that some memory is logically uninitialized for some section of the program, then it may be able to reuse that memory for other things (e.g. temporaries) without having to worry about messing up "real" state. If the compiler sees a branch, but one of the paths would access uninitialized memory, it *may* be able to assume that the program never takes that path, since that path is going to have trouble doing something useful.

A lot of focus is given to the optimization side of uninitialized memory, because when it breaks badly for you it *sucks*. People quickly jump to arguing that all memory should be default-initialized (0 is a good value, everyone loves 0!), or that the language should be "lower level" and actually let you access uninitialized memory because you are Very Smart and Know What You're Doing™.

But I want to push back on that a bit, because the checking function is *really valuable*! I get it, this can feel laughable to someone scarred by uninitialized memory bugs in C++. Am I seriously arguing that "uninitialized memory helps you write correct programs"? I am! What I *would* argue is that C++ doesn't bother to do that work. (By default, tools like lints and ubsan *are* using the concept to help you!)

You don't need to build a system where the programmer can just access obviously uninitialized memory. You can build a system that statically determines that memory is uninitialized (or *maybe* uninitialized), and produces an error. The section on Definite Initialization will give you lots of simple examples of this.

Sure, you can build a system that default-initializes every value to 0 and it's *fine*, but it's also giving up on an opportunity to help the program write a program that does something useful. If everything is always initialized to 0 and it's ok for programmers to rely on that, you can never distinguish between "actually wanted 0" and "forgot to put a useful value there". You're saving like, 3 keystrokes in exchange for the ability to catch really nasty bugs.

That might be a genuinely reasonable choice for your language, but it's not an *obvious* one. It's also not one you need to make in such a binary manner! You could make it a *warning* to rely on the default value, but happily run the program anyway. Let the programmer test out their program, but reserve the ability to suggest ways it can improve. In this way you are basically saying the value is still *logically* uninitialized, but the compiler will do its best to try to do something reasonable about it.

I think there's a lot of really interesting potential in languages that can "do what I mean" but actually tell you about it and help you rework your program into something more robust!



### Expanding The Scope of Uninitialized Memory

When I think "uninitialized memory" my mind immediately goes to a fresh allocation that hasn't had a value written to it, and I expect this is what most other people picture as well. But we don't need to limit ourselves to that!

Many languages (or at least their implementations) have the concept of a variable becoming uninitialized *after* it has been initialized, requiring it to be reinitialized. This can happen when the value in the variable is "destroyed" or "moved". These kinds of operations may not actually *modify* the underlying memory, but they still semantically reset all the bits to `uninit`.

For instance, if you have a pointer into the heap and free it, the underlying memory may still contain the freed pointer, but it's not correct to treat that memory like a valid pointer anymore. A freed pointer may as well be random garbage, so it's reasonable to think of this memory as uninitialized again.

We can also consider something "uninitialized" at one level of our design, but at lower levels regard it as initialized. In this way we can use the *concept* to help us think about program correctness, without making the stakes "if you mess up we will miscompile your entire program".

For instance, many languages have the notion of *sentinel values* which are basically just values that mean "there is no meaningful value here". See: `null`, `Option::None`, `NaN` (floating point), and `undefined` (JavaScript). C++ also arguably does with many "default" values (i.e. `std::unique_ptr`'s default constructor is just `nullptr`).

Sentinel values let the programmer *logically* work with uninitialized memory without the nondeterminism and Undefined Behaviour traditionally associated with "real" uninitialized memory. The programmer can query if something is "uninitialized", or even explicitly mark it as "uninitialized". And if they accidentally "use" it as if it *wasn't* a sentinel value, then ideally something reliable will happen.

Popular strategies for handling "uses" of sentinels include:

* Immediately panic and crash the program. Simple and effective, but it sure sucks when it happens.

* Discard the requested operation and replace its result with the sentinel. `NaN + 5` in floating point is still `NaN`. `[nil someMethod]` in Objective-C (calling a method on null) is a noop that *usually* returns 0 (depends on type, not getting into it).

* Make them silently coerce into a more appropriate value for the operation. Sentinel values being "falsey" is super popular, but they can also be "zeroish" or "NaNish" or anything else if you really just want to try to make the program do *something vaguely reasonable*.




### Freeing Pointers: Spooky Deinitialization At A Distance

In the previous section we saw that when a pointer becomes freed it *isn't* mutated but its semantics change. Namely, it essentially becomes useless random garbage, and is reasonable to think of as uninitialized memory that shouldn't be used anymore.

The problem is that on its own, this isn't a terribly useful insight. Pointers can be freely copied and offset, and all those "derived" pointers are arguably now also uninitialized memory (they're dangling). It's really easy to know the location of the pointer that was actually passed into `free`, but it's a lot harder to know where all those other derived pointers are. 

What use is this horrible Spooky Deinitialization At A Distance?!

Remember, there are two functions of thinking about uninitialized memory: checking and optimization.

From the perspective of *optimization* this limitation can be ok -- if we lose track of a derived pointer, then at worst we're missing an optimization opportunity. 

From the perspective of *checking* this is an absolute nightmare. There's a very good reason that almost every programming language is garbage collected! But if we don't want to implement a garbage collector, it would be good to come up with a system that can help us make sense of the Spookiness and help people write correct programs.

And that's how you end up with Rust's Borrow Checker. Just implement that. End of Article, Draw the Rest Of The Owl.

Ok but seriously, Rust's whole "ownership" and "borrow checking" thing is basically the logically conclusion of trying *really* hard to make it possible to do whatever with pointers and still be able. Pointers are complicated.

That said, we don't necessarily need to be as *Extra* as Rust to still get some reasonable stuff done. You can build a simpler system that handles most cases reasonably. Then you can either force the programmer to constrain themselves to this system, or design a fallback system that keeps things working reasonably.

We'll discuss that in the next section.



## Background Concept 2: Ownership

I will mostly be using Rust terminology for these concepts, because I like them (I'm biased and this is my article). More seriously I think Rust does the best job of keeping these concepts clear, because it expects programmers to think about them a lot more than any other language. The one exception is that I'm going to use "bitcopy" for the kind of shallow copy rust calls Copy, because the word "copy" is way too overloaded of a term in programming for me to just throw around.

The system we will be considering is called *ownership*. You can think of it in a few different ways, and use it for different things, but this article is all about uninitialized memory, so we'll be thinking about it a lot in this regard. That said, you can also think of it as a system for tracking and controlling the flow of values through a program.

A system for managing the flow of values through a program is so useful because of the issue discussed in the previous section: *Spooky Deinitialization At A Distance*. If we know how values flow through our program (either by tracking them or restricting their flow), then we can know all the things that suddenly are (or could) be Spooky Deinitialized.

You can see aspects of ownership in the user-facing semantics of Rust, C++, and Swift. You can also see aspects of ownership in lots of a compiler backends, because wow it's just really useful for optimization and analysis.

The first step in understanding ownership is to make a distinction between types which are "plain old data" (POD) and those that aren't. POD types are "trivial" in the sense that they can't be invalidated. They're just a pile of bytes with a simple interpretation, and there isn't any concern for Spooky Deinitialization. Primitives (Integers, Floats, Bools, ...) are generally POD. A composite type that's just "a bunch of primitives" such as `Vec4(Float, Float, Float, Float)` is also POD.

Anything you're more concerned with and want to really precisely track the flow of (such as heap pointers that might be freed), you make non-POD. This is a very cumbersome term, so **for the rest of this article all types and values will be non-POD unless stated otherwise.**

Instead of treating these values as random piles of bytes, we will regard them as distinct instances that **never exist in more than one place in the program**. That's a strong restriction, but that's why this system can be fairly simple!




### Ownershiplang: Pseudocode and Terminology For Ownership

To help us discuss ownership and look at examples, we will be looking at a lot of examples using a "pseudo-Rust" language that I am going to call Ownershiplang. Beyond the usual if/else/while stuff, it has the following operations and terminology.

**QUICK REFERENCE**

* `let x = MyType()`: call MyType's constructor and init a new var x with it
* `relet x = MyType()`: call MyType's constructor and reinit an existing var x with it
* `drop x`: run destructor of x and deinitialize the variable
* `bitcopy x`: bitwise (shallow) copy x
* `move x`: bitwise (shallow) copy x, and deinitialize the variable
* `clone x`: deep copy x 
* `y.clone_from(&x)`: update `y` to be a clone of `x` without dropping `y` 

A **variable** is any location in memory, but by default we will assume they are normally scoped local (stack) variables. Variables may be initialized or uninitialized. We say that initialized variables *own* their values, because they're the only location in memory where that *specific* instance can be found. Variables *logically* don't share memory locations with any other variable (but the compiler can still merge them as an optimization).

When we **construct** a value we create a new instance of its type (call the type's constructor), and initialize a variable with the value. Generally we will just write this as `x = MyType(...)`. To be extra clear, we will write `let x = MyType(...)` when we are creating the variable at the same time, and `relet x = MyType(...)` when we are *reinitializing* an uninitialized variable. Assignment without `let` or `relet` will be intentionally ambiguous, and we will discuss how that surface syntax in our language might be "desugarred".

When we **drop** a variable, we run the destructor of the instance stored in that variable (destroying the instance forever) and deinitialize the variable. Ideally all instances get dropped before the end of the program, but this is generally only "best effort" (can't do anything if the computer gets unplugged!). Most importantly we must *never* drop an instance multiple times. That's a recipe for double-frees.

When we **bitcopy** a variable, we just copy the bits to another variable. The destination variable is initialized, and the source variable remains initialized. This is only ever semantically valid for POD types, so we won't see bitcopy much, but it's useful to define to properly understand how *move* is different.

When we **move** a variable, we bitcopy it *but also deinitialize it*. This does exactly what it says on the tin: the instance is logically moved from one location to the other. You could also say that ownership is transferred from one variable to the other.

When we **clone** a variable, we create a new instance that's a deep copy of the value. So generally if the variable owns a heap-allocated pointer, the new instance will have its own fresh heap-allocated poitner. This is what C++ calls a "copy-constructor".

To better describe the semantics of C++, it is also useful to have the notion of `y.clone_from(&x)`. This operation does not create a new instance, but rather updates `y` to be equivalent to a clone of `x`. This removes a constructor-destructor pair from our program, and potentially reuses some resources (although it might just end up destroying and recreating all of the contents of `y` anyway). clone_from is what C++ calls a "copy-assign operator".

You could consider similarly defining `y.move_from(&x)` to try to describe "move constructors" in C++, but this is a lot muddier. Conceptually we would expect "move_from" to deinitialize the source variable, but that isn't *quite* how C++ works. Usually the source variable is reset to the type's "default" value, which is often a sentinel. Previous sections argue that sentinels are *logically* uninitialized, so that's *basically* an ownership-style move, but it's different enough that I don't want to muddy the waters with it.

For examples I will generally use a "pseudo-Rust" language. We will view it both in a "sugarred" form which reflects the surface syntax a programmer might write, and a "desugarred" form, where we try to determine what that surface syntax actually means. When desugarred, I will try to use the right-hand-side to annotate how the initialization state of each variable changes.

(You can easily distinguish ownershiplang and Rust based on whether statements end with semicolons. Ownershiplang has none, because parsing is for nerds.)

So given this surface ownershiplang program:

```rust,ignore
let x = MyType()
let y = x
x = clone y
```

We might desugar it as the following:

```rust,ignore
let x = MyType()    // x init
let y = move x      // x init -> uninit, y init
relet x = clone y   // x uninit -> init, y init
drop y              // x init          , y init -> deinit
drop x              // x init -> deinit, y deinit
```

Note how we automatically insert `drop` calls for initialized variables that go out of scope. Here it was easy, but will it always be?




### Ownership: What's The Point?

Ok so I've defined all this stuff about "ownership" and "flows of values", but what's the point?

Well in the most trivial sense, values that obey ownership are trivial to track: they are exactly where they are and nowhere else. If you know a variable is initialized and the type of its value, then you know how to destroy it (just drop it), and you can be confident there won't be any Spooky Connected Copies floating around.

If you do need to "copy" a value, you *clone* it, which creates a totally independent instance. In this sense you just don't need to *care* about the flow of derived values, because whenever a value gets "copied" both versions are totally independent instances.

Let's look at [how ownership is used in Swift](https://gankra.github.io/blah/swift-abi/#ownership) to better understand what this can do for us.

Swift's design leans *very* heavily on its Automatic Reference Counting (ARC) system (which interestingly doesn't have a cycle collector, you need to manually break cycles with weak pointers).

Although Swift doesn't generally expect developers to *think* about ownership, all these reference counted pointers are ultimately governed by ownership. In its parlance a move is a +0 operation and a clone is a +1 operation, and a drop is a -1 operation. Because that's what ownership *is* for reference counted pointers: just determining how the reference count needs to be adjusted.

Different kinds of functions define in their ABIs whether they move or clone their various arguments (including `self`). Note that you can define a temporary "borrow" of a value as two moves: you move it to the borrowed location, and then move it back. So it's +0 but "be really careful" because the original location is logically uninitialized during this time. This is why [borrows are implemented as callbacks and coroutines in swift](https://gankra.github.io/blah/swift-abi/#materialization).

Having this ownership system to lean on makes it much easier to reason about how different operations need to adjust reference counts, and how everything composes.

Having this really robust system for managing reference counts is *especially* important in Swift because of one of it's other major tricks: implementing value semantics with Copy on Write (COW).

Types that have value semantics (structs in Swift) are supposed to behave kinda like POD types. They don't alias eachother, and when you assign them to a new location, it creates a clone of the value and both locations are left initialized. This can be *incredibly* expensive when heap allocations are involved -- either because the language needed to box the struct up, or because the struct contains a heap allocation it owns (collections like Array and Dictionary all have value semantics!!!).

So to smooth over the performance problems, Swift applies a clever little trick: you can actually reference count structs and let them "alias" *as long as you clone them before an aliasing mutation*. As it turns out, this actually totally preserves value semantics! To convince yourself of this, note that this is basically just deferring all the clones until they're needed. 

Specifically, whenever you try to perform an operation on a struct that could mutate it, Swift checks if the reference count is greater than 1, and if it is, it clones the struct and creates a new reference counted pointer for that variable (memory location) that can't be aliased yet.

This works, but it also gives Swift some extremely Fun performance hazards. For instance, if you're modifying an Array in a loop and accidentally bumping the reference count, your `O(n)` algorithm can suddenly be `O(n^2)` (because a reference count bump *is* semantically cloning it would have been quadratic even without COW, but COW "handling it for you" all the time means that copies might not be obvious).

So *wherever* Swift can find a place where a refcounted pointer can be moved (+0) rather than cloned (+1) it's not just a matter of saving an atomic increment, it can be the difference between linear and quadratic!

Ownership: it's damn useful!






## Background Concept 3: Definite Initialization Analysis

The purpose of this section is that I will frequently say "if you use Definite Initialization then you can...". So here's what that means and what it can do.

*Definite Initialization* (DI) analysis can be found in many production compilers, and it basically tracks how the initialization of variables flows into the uses of the variables.

If you have ever seen a compiler spit out a warning to the effect of "hey you're assigning a value here but nothing ever reads it", that's probably some version of DI. For instance, here's rustc doing it:

```rust
fn main() {
    let mut x = 0;
    println!("{}", x);
    x = 1;
    println!("{}", x);
    x = 2;
}
```

```text
warning: value assigned to `x` is never read
 --> src/main.rs:6:5
  |
6 |     x = 2;
  |     ^
  |
  = note: `#[warn(unused_assignments)]` on by default
  = help: maybe it is overwritten before being read?
```

Basically DI goes through every point in a program and tracks whether the variable is
initialized at that point in the program. In its simplest form, the possible values 
are "(definitely) uninit", "(definitely) init", and "maybe (init)".

So when a variable is declared (or deinitialized) it has the value "uninit",
when it's assigned to it has the value "init", and whenever two paths in
the program merge back together that don't agree, it becomes "maybe".

(Being able to warn about unused values requires you to also track "definite usage"
in a similar way that will be left as an exercise to the reader.)

For example, the compiler might see the program like this:

```rust,ignore
let x: u32;             // uninit
                        // uninit
if condition {          // uninit
    x = 3;              // init
    println!("{}, x");  // init (use is valid!)
}                       // maybe
                        // maybe
println!("{}", x);      // maybe (ERROR)
```

If you have this analysis, you can do more interesting things with initialization. Rust is a really nice demo of this because its whole schtick with ownership and borrowing depends a lot on Definite Initialization, so it actually lets you do some non-obvious stuff as long as DI can prove it's ok. 

For instance, all of these programs compile and run without warnings:

```rust
fn main() {
    // Rust does not default initialize ints, `x` is uninit,
    // and we also haven't marked the variable as mutable!
    // That's ok as long as we always init it before using it,
    // and never reinitialize it.
    let x: u32; 

    if true {   // Rust does not abuse the fact that `if true` is trivial!
      x = 3;
    } else {
      x = 4;    // Rust lets us assign to x even though it's immutable
                // because it knows it's Definitely Uninitialized here! 
    }

    println!("{}", x); // Rust knows x is Definitely Initialized here!
}
```

----

```rust
fn main() {
    // It's ok to not initialize `x` in every path, as long as
    // we only access it in places where it's Definitely Initialized.
    let x: u32;

    if true {
        x = 3;
        println!("{}", x); // Can access x here, it's definitely init!
    }

    // Can't access x here, it might be uninit!
}
```

----

```rust
fn main() {
    // Neat trick that lets you "return" a reference to a value 
    // even though you're conditionally computing it in an inner scope.
    let temp;
    let my_borrowed_string;

    if true {
        temp = format!("computed string {}", "wow");
        my_borrowed_string = &*temp; // Ok! temp is init and outlives the scope
    } else {
        my_borrowed_string = "static string!";
    }

    // At this point we can't access `temp` because it's might be uninit
    // but `my_borrowed_string` is definitely init!
    println!("wow! {}", my_borrowed_string);
}
```

----

```rust
fn main() {
    // DI even understands loops!
    let x: u32;    // Still immutable!

    loop {
        if true {
            // DI understands this block is only entered once, because
            // it ends with a `break` that ends the loop.
            x = 5; 
            break;
        }
        // Assigning to x here would be an error, 
        // as it may happen multiple times!
    }

    // DI understands that to get out of the `loop`, you must have taken
    // the path with a `break`, so x must be definitely init here!
    println!("{}", x);
}
```

-----

Meanwhile, these programs fail to compile:


```rust,ignore
fn main() {
    let x: u32;

    println!("{}", x);
}
```

```text
error[E0381]: borrow of possibly-uninitialized variable: `x`
 --> src/main.rs:4:20
  |
4 |     println!("{}", x);
  |                    ^ use of possibly-uninitialized `x`

For more information about this error, try `rustc --explain E0381`.
```

------

```rust,ignore
fn main() {
    let x: u32;

    if true {
        x = 3;
    }

    println!("{}", x);
}
```

```text
error[E0381]: borrow of possibly-uninitialized variable: `x`
 --> src/main.rs:8:20
  |
8 |     println!("{}", x);
  |                    ^ use of possibly-uninitialized `x`

For more information about this error, try `rustc --explain E0381`.
```

------

```rust
fn main() {
    let x: u32;

    if true {
        x = 3;
    }
    
    x = 4;
    println!("{}", x);
}
```

```text
warning: value assigned to `x` is never read
 --> src/main.rs:5:9
  |
5 |         x = 3;
  |         ^
  |
  = note: `#[warn(unused_assignments)]` on by default
  = help: maybe it is overwritten before being read?

error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:8:5
  |
2 |     let x: u32;
  |         - help: consider making this binding mutable: `mut x`
...
5 |         x = 3;
  |         ----- first assignment to `x`
...
8 |     x = 4;
  |     ^^^^^ cannot assign twice to immutable variable

For more information about this error, try `rustc --explain E0384`.
```































# Problems and Problematic Features

So for the bulk of this post I will be focusing around building a C-like (really Rust-like) language with *constructors* and *destructors* -- something like C++, Rust, and Swift. Even those 3 languages have a lot of really complicated differences in how they handle initialization and deinitialization! (I worked on Rust and Swift, and have reasonable knowledge of C++, but I sometimes mix up the details, it's really complicated and subtle!)

Some of the ideas here will be applicable to languages that don't have "constructors" and "destructors", because really what we're talking about is thinking about the flow of values and initialization state in a program. Even if you build a nice dynamic automatically reference-counted language, a lot of these *ideas* are there under the hood -- creating a reference counted object is like running a constructor, bumping the reference count is a clone (copy constructor), reducing a reference count is like dropping (destructor).

















## Problem: Constructor Transactionality

The user has defined some type like this:

```
struct MyType {
    x: Int,
    y: Box,
}
```

Because it contains multiple values, it can't "atomically" be created, so the program can be in a "zombie" state where a value is *partially* initialized. You don't want people to access the uninitialized parts, or to forget to initialize some of them -- you want constructors to be *transactional* in some way -- either it all happens or none of it happens, and you can't mess around with the intermediate state.

I know of 3 approaches to handle this:

1. Initialize everything to a default value (Java)
2. Force the user to provide the values for every field at once (C++, Rust)
3. Use Definite Initialization to verify all fields are initialized (Swift)

C++ and Rust both basically require you to have a specific point in your program where you provide all the values for the fields at once, to minimize the scope of this zombie period. In this way the value is *semantically* constructed atomically, because there's no point where the value can be referred to but isn't fully initialized.

Swift instead pulls out our good friend Definite Initialization and just requires all the fields to be initialized before you can access the whole value.



### Java Constructors

In Java, before the first line of a constructor even executes, the runtime has already filled in default values for every field, so the type is already completely initialized without you having to do anything. Certainly simple an effective!

Having everything default initialized can be a bit wasteful, but optimizers exist for a reason. Perhaps more interestingly, this implicitly forces everything in Java to *have* a default value. Not a terribly difficult requirement given the fact that almost every type is actually a pointer which can be null. (I think every type's default value is "0" at the bit-level, but don't quote me on that.)

(I bet there's some interesting stuff to say about C# here, but I wouldn't know it!)

```java
class MyType {
    int x;
    Integer y;
      
    MyType() {
        // All fields have already been initialized to default values.
        // Can already access `this` because it's already fully initialized
        System.out.println(this);   // MyType@some_address
        System.out.println(this.x); // 0
        System.out.println(this.y); // null
        
        // Can set our own values, or not, doesn't matter
        this.x = 3;
        this.y = 5;
    }      
}
```



### Rust Constructors

Rust uses "record syntax" to force you to provide all the values for the fields as one ""atomic"" step. Constructors are pretty boring in Rust, but I'm including them here because they're interesting *in contrast* to what C++ does (see below).

```rust
struct MyType {
    x: u32,
    y: Box<u32>,
}

impl MyType {
    // Constructor functions are actually just a convention in Rust.
    // They're just normal static functions that happen to return Self.
    fn new() -> Self {
        // The "real" constructor is this record syntax, which can
        // only be used if you are allowed to access all fields.
        let result = MyType { 
            x: 3, 
            y: Box::new(5), 
        };
        // value is full initialized here, can be used like normal
        
        result // return it at the end
    }
}
```



### C++ Constructors

C++ uses initializer lists (is that the right term?) to force you to provide all the values for the fields as one ""atomic"" step. 

Although on the surface this is fairly similar to what Rust does, with C++ you have access to your `this` pointer in the constructor, which implies that the location in memory for the C++ value has already been allocated and passed in to the constructor.

This is very important in C++ because it puts a lot more weight on instances of a type being tied to a specific location in memory. This is why "copying" or "moving" a value is described as a "constructor" -- the change of location logically requires a new value! This is *very* important for values which contain "intrusive" pointers into themselves. 

As a result, C++ is much better at intrusive pointers than Rust. Rust absolutely hates intrusive pointers (although there's plenty of Fun Tricks to deal with it, the most common just being "store array indices instead of pointers into the array"). A more C++-like design is certainly worth considering if you care a lot about intrusive pointers!

In *theory* this also means C++ should be better at avoiding copies than Rust, because Rust just kinda slops values around and hopes the copies will optimize out, while C++ *semantically guarantees* that values are constructed in place.

This is only in theory because the existence of implicit copy/conversion constructors means [it's actually incredibly easy to accidentally *deep copy* (clone) values all over the place](https://groups.google.com/a/chromium.org/g/chromium-dev/c/EUqoIz2iFU4/m/kPZ5ZK0K3gEJ), which is wildly more expensive than the shallow copies Rust is prone to. Anecdotally, I've found this means that Rust wins out when you're not thinking terribly hard about copies and allocations, but Your Mileage May Vary.

```cpp
#include <memory>

struct MyType {
    int x;
    std::unique_ptr<int> y;
    
    MyType(): 
        // All fields initialized here ""atomically""
        // Is this the most best way to set up a unique_ptr member? I don't care.
        x(3), y(std::make_unique<int>()) 
    {
        // `this` is full initialized here and can be used like normal
        // Although in this case we still have some work to do.
        *this->y = 5;
    }
};
```



### Swift Constructors

Swift uses Definite Initialization to force you to provide all the values for the fields before ending an initializer or calling methods on `self`. It's neat!

It's proper DI too, you can conditionally assign fields and do whatever as long as things are Definitely Initialized by the end! Swift also has "convenience initializers" (AKA "delegating initializers") to distinguish between initializers which *really* initialize the values, and those that just call into another initializer.

Interestingly, convenience initializers *can* call other convenience intializers -- you just need to bottom out in a "designated" initializer that actually does the initialization. There is no checking for this, so you can actually just infinitely mutually recurse and blow the stack if you really want to! 💯

Although when class inheritance is involved you must specifically call a non-convenience ("designated") initializer of the superclass. I don't *fully* grok all the constraints here, because there's a lot of weirdness going on around subclasses being able to override things and some really weird legacy Objective-C patterns. 

Basically a subclass is actually allowed to override some initializers of the superclass, and the superclass *will* actually call them? And that's useful sometimes? For reasons? But the designated intializers are opted out of this system so you don't go around in infinite loops up and down the inheritance chain and mess up all the DI logic.

A bunch of Swift developers tried to explain this to me and it felt like my head was going to explode. I think the entire discussion was best summarized by one of them saying "I think for this to make the *most* sense you have to put yourself in a Smalltalk mindset". I can't.

```swift
class  MyType {
    var x: Int32;
    var y: MyBox<Int32>;

    // Designated ("real") initializer which must initialize all fields
    init() {
        // At this point you *can't* access `self` as a full
        // value, read any fields, or call any methods.
        self.x = 3;

        // You can access fields that are Definitely Initialized.
        // When `self` is only partially initialized!
        print(self.x);

        self.y = MyBox(5);

        // If we had a superclass, we would initialize it here

        // *now* the compiler knows you're Definitely Initialized
        // and will let you return or use self like normal.
        print(self)
    }

    // Convenience ("fake") init, which must call another initializer.
    convenience init(sayImCool: ()) {
        // Illegal to even try to initialize self.x here because
        // a designated initializer assumes all fields are 
        // definitely *uninitialized*, and that would mess it up!
        // (At best, destructors could get leaked.)
        print("you're so cool!")

        // Call the designated initializer
        self.init()
        // Now self can be used normally
    }

}

class MyBox<T> {
    var val: T;
    init(_ newVal: T) { self.val = newVal; }
}

```























## Problem: Reassignment

So given `MyType` with a destructor, you can write this:

```rust,ignore
let x = MyType(1);
x = MyType(2);
```

And what exactly that does can be... surprisingly complex. 

Let's not worry about statements like the first line, where a fresh variable
is declared and assigned all at once. We can reasonably convince ourselves
that this is a "transactional" operation.

Let's instead focus on expanding out the operations with 3 operators:

* drop val -- call the destructor of `val` and deinitialize the variable
* clone val -- deep copy `val`
* move val -- move the contents of `val` out, making `val` uninitialized

(shallow copying isn't really a coherent concept for types with destructors,
because this can result in things like double-frees.) 

Here are some potential expansions:



### Reassignment Desugar 1: drop-construct

Drop x before reinitializing it with the new value.

```rust,ignore
let x = MyType(1)   // x init
drop x              // x init -> uninit
relet x = MyType(2) // x uninit -> init
drop x              // x init -> uninit (but goes out of scope)
```

This is in some sense "optimal" and what I intuitively picture when I see this
code. However it has a problem: there is a non-trivial period of time where
`x` is in scope, but not initialized (the period when `MyType(2)` is executing).

If `x` can be referenced (including with a closure!), then things are very
wonky. It's *pretty* reasonable to update a value using itself, so you probably
don't want it to get destroyed before computing its replacement. 

Otherwise this potentially reasonable code will be a Problem:

```rust,ignore
let x = MyType()  
x = MyType(&x)    // bitcopies some POD values
```

Remember, we can model "borrowing" as "move in, move out" so using this desugarring we would get:

```rust,ignore
let x = MyType()      // x init
drop x                // x init -> unit
??? = MyType(move x)  // ERROR: use of uninitialized value "x"
```

(I omitted the move out part because everything breaks before you get to that step and both values want to assign to the same location so it's extra confusing.)

Desugar 2 has a more reliable version of this operation, but you could imagine using ownership to determine if you can "get away with" using the more efficient Desugar 1. This is certainly possible, but it might make the language a bit more inconsistent. There's an argument for sticking with Desugar 2 for consistency's sake.



### Reassignment Desugar 2: construct-drop-move

Store the new value in a temporary, drop x, and then move into x.

```rust,ignore
let x = MyType(1)     // x init
let temp = MyType(2)  // x init          , temp init
drop x                // x init -> uninit, temp init
relet x = move temp   // x uninit -> init, temp init -> uninit
drop x                // x init -> uninit, temp uninit

// temp is not dropped because it was moved into x!
```

As far as I can tell, this is how Rust works (I was *sure* it did Desugar 1, dang!). It's not *actually* that much more inefficient than Desugar 1, especially when optimizers get involved, but well, there's a reason Rust has a reputation for codegen riddled with memcopies!




### Reassignment Desugar 3: construct-clone_from

Store the new value in a temporary, and use clone_from to try to minimize the
cost of cloning instead of moving.

```rust,ignore
let x = MyType(1)     // x init
let temp = MyType(2)  // x init          , temp init
x.clone_from(&temp)   // x init          , temp init
drop temp             // x init          , temp init -> uninit
drop x                // x init -> uninit, temp uninit
```

To the best of my understanding, this is how C++ does it. Remember, C++ has a really strong notion of tying values to their location. It basically doesn't want to use what we are calling move, because that would change the location of a value!

C++ also doesn't really like the concept of "reconstructing" a value. It would much rather you update the existing value in-place. Hence why we end up using clone_from. 

Note that we don't get an extra temporary when we first define `x` because the initial declaration of a variable is *special*. Because the variable has *never* contained an initialized value at this point, C++ knows there's no point in doing any kind of temporary dance and can construct the value in place.

Looking at the initialization state in our desugarring, you can see how things are in some sense a lot cleaner and nicer: nothing ever becomes uninitialized until it goes out of scope. This isn't really a tenable solution for something like `vector<MyType>` though, so there *is* machinery for destroying a value and constructing a new one in that location.













## Problem: Delayed Initialization

(Delayed initialization is actually the one place where it's worth talking about POD values, because even an integer can be uninitialized *at first*, but there's no need to consider them differently from non-POD values, so we'll stick to non-POD.)

Here's a simple example of delayed initialization you *might* want your language to support:

```rust,ignore
let x
let y = MyType(1)
x = MyType(2)
```

So what does this actually *mean* in our language? How do we actually *analyze* and *implement* this? There are a lot of options, and we'll try to look at them all in "Implementing Ownership" section. For now we can look at if these examples make sense from the perspective of ownership.

```rust,ignore
let x                 // x uninit
let y = MyType(1)     // x uninit        , y init
relet x = MyType(2)   // x uninit -> init, y init
drop y                // x init          , y uninit -> init
drop x                // x init -> uninit, t uninit
```

Seems fine, all makes sense from the perspective of ownership.

Let's make it a bit more complex, and introduce some branches:


```rust,ignore
let x

if cond {
    x = MyType(1)
} else {
    x = MyType(2)
}
```

Well if our analysis (*whispers* Definite Initialization...) can handle merge the results of branches together, then we can still statically make sense of this:

```rust,ignore
let x                   // x uninit
                        // x uninit
if cond {               // x uninit
    relet x = MyType(1)     // x uninit -> init
} else {                // x uninit
    relet x = MyType(2)     // x uninit -> init
}                       // x init (merged "init" and "init" from both sides)
drop x                  // x init -> uninit
```

But what if we change this example a little bit?

```rust,ignore
let x

if cond {
    x = MyType(1)
}
```

Now we aren't initializing x so cleanly:

```rust,ignore
let x                   // x uninit
                        // x uninit
if cond {               // x uninit
    relet x = MyType(1)     // x uninit -> init
} else {                // x uninit
                            // x uninit 
}                       // x MAYBE (merged "init" and "uninit" from both sides)
???drop??? x            // x MAYBE
```

This is what Rust calls *dynamic drop*. If you want this program to compile, it literally can't be exclusively handled by static analysis (sorry Definite Initialization, I still love you!!!).

We'll look at strategies for handling this in the "Implementing Ownership" section.







## Problem: Moving Out

Just like you might want to support delayed initialization in your language, you might want to also support its natural counterpart: moving a value out of a variable before it goes out of scope.

Once again, actual implementation details can be found in the "Implementing Ownership" section.

First let's have a minimal example of moving out of a variable:

```rust,ignore
let x = MyType(1)
consume(x)
```

Which can be expanded to this:

```rust,ignore
let x = MyType(1) // x init
consume(move x)   // x init -> uninit
                  // x uninit

// No drop at end of scope, x is uninitialized
```

Once again, Definite Initialization will help you track this in the simple cases, but things quickly get more complicated:

```rust,ignore
let x = MyType(1)

if cond {
    consume(x)
}
```

Which can be expanded to this:

```rust,ignore
let x = MyType(1)       // x init
                        // x init
if cond {               // x init
    consume(move x)         // x init -> uninit
} else {                // x init
                            // x init 
}                       // x MAYBE (merged "uninit" and "init" from both sides)
???drop??? x            // x MAYBE
```

Just as with Delayed Initialization, we need Dynamic Drop. And I won't put it off anymore, it's time to talk about Implementing Ownership!






# Implementing Ownership

We've talked a lot about the ideas in ownership and how they can let us reason about the flow of values in our program, and how we can "desugar" some higher-level concepts to ownership semantics. But what do we even *do* with this information?

And what do we do about the weird cases we saw in the previous section where things are only "maybe" initialized?

Well, we have a lot of different options! First off, we're going to look at a few different versions of *sentinel values*. If you don't want to build any fancy static analysis, sentinel values are the solution for you!

The idea is to have runtime values which mean "there's no useful value here" or "it's empty", anything really that your `drop` implementation can quickly look at and go "nothin' to do".

We'll be looking at 3 slightly different approaches to this:

* null
* empty values (C++-like)
* smuggled drop flags (early versions of Rust)





## Ownership Strategy: Null Sentinels (And Reference Counting with Moves)

Lots of languages very quickly come to the conclusion that things would be *a lot* easier if everything was more uniform, and quickly come to the "everything is a pointer" strategy. And if everything is a pointer, you get a "there's no value here" value for free: `null`!

Let's consider the design of a reference counted language (similar to Swift) that implements ownership and moves by using sentinel values.

Now I imagine you're thinking: hey what the heck, this is just basically every programming language ever! And you're not completely wrong, but the difference is how we manipulate pointers at runtime.

Consider this simple code:

```rust,ignore
let x = alloc_refptr MyType(1)
let y = x
```

We want to drop down to thinking about our actual implementation, so we're going to go down to lower-level semantics, which everyone knows you do by putting `%`'s before you variable names. Because we're looking at the low-level implementation of reference counting, all of our variables now hold POD pointers. So `%y = %x` just means "copy the bits of %x into %y".

Here's what a normal reference-counted language with "copy" semantics would do:

```text
%x = alloc(MyType_size)  // Allocate the pointer with refcount 1
MyType_init(%x)          // Run the constructor to initialize the memory
increment_refcount(%x)   // refcount => 2
%y = %x                  // y and x now point to the same location
drop(%y)                 // y scope ends, refcount => 1
drop(%x)                 // x scope ends, refcount => 0 (MyType_destructor)
```

But here's what a language that uses "move" semantics would do:

```text
%x = alloc(MyType_size)  // Allocate a refcounted pointer with refcount 1
MyType_init(%x)          // Run the constructor to initialize the memory
%y = %x                  // y and x now point to the same location
%x = null                // x is now marked as "uninitialized"
drop(%y)                 // y scope ends, refcount => 0 (MyType_destructor)
drop(%x)                 // x scope ends, noop because x is null
```

As you can see, we can implement "move" in our language by just having it not touch the reference count at all, and instead set the source variable to `null`. We also don't need to do any special handling for `drop` when a variable becomes "uninitialized" -- we just unconditionally drop a variable when it goes out of scope, and `drop` starts by checking if it's `null`! 

As much as I love talking about Definite Initialization as the panacea that will solve all of our problems, you really don't *need* it for move semantics. You can just handle it one operation at a time like this, and everything will "just work" (caveat: you need to be very mindful of unwinding if your language has that).

This system can also implement dynamic checking and handle the nasty cases we weren't sure about, although exactly how we want to do that depends on what we want to support. 

For instance, this program is something that even Rust would reject, because there is a possible path of execution where we try to `consume` an uninitialized x. But you could certainly imagine wanting to build a language that allows this to work if the execution makes sense at runtime!

```rust,ignore
let x

if cond1 { 
    x = MyType(1)
}

if cond2 {
    consume(move x)
}
```

In the extreme, we could have absolutely no analysis and generate totally mindless and bullet proof code like this:

```text
%x = null                    // x uninit

if cond {
    %temp = alloc_refptr(MyType_size)   // allocate a new refptr
    MyType_init(%temp)                  // initialize it
    drop(%x)                            // drop x in case it was initialized
    %x = %temp                          // now "move" into x
    %temp = null                        // mark temp as uninitialized
    drop(%temp)                         // drop temp in case it's initialized
}

if cond2 {
    %consume_args = ...                 // <some calling convention shit>
    
    assert_nonnull(%x)                  // trying to use x, crash if uninit
    %consume_args.arg0 = %x             // move x into the argument slot
    %x = null                           // mark x as uninit

    consume(%consume_args)              // call the function
}

drop(%x)                                // drop x in case it's initialized
```

There's a lot of extra code blindly setting stuff to null or checking if it's null, but hey this means your codegen doesn't need to have *any* kind of higher-level analysis of the program. All it needs to know is when variables go out of scope.

There are however a few simple "local" cleanups you could apply to improve this.

Since you "know" %temp is a temporary that your codegen is creating just to move out of it, you can just not emit the cleanup code when you create it:

```text
if cond {
    %temp = alloc_refptr(MyType_size)   // allocate a new refptr
    MyType_init(%temp)                  // initialize it
    drop(%x)                            // drop x in case it was initialized
    %x = %temp                          // now "move" into x
    
    // program implicitly forgets temp exists
}

```

If you're fine with runtime errors being more "late bound" you don't actually
need to check if the a moved variable is "initialized" or not. You can choose
to only check this when it really matters (i.e. when accessing a field of MyType). 

```text
if cond2 {
    %consume_args = ...
    
    // If x is uninitialized (null), we just pass that in as an arg.
    // This eventually will blow up if %x is *really* accessed.
    %consume_args.arg0 = %x

    // null => null, no harm no foul
    %x = null

    consume(%consume_args)
```

And of course, having a sentinel value means that you can let the *programmer* use it and query it at runtime, if you really want. But you don't *have to*, all this null stuff could be a secret implementation detail that the programmer has no access to.

Now you may be wondering *what on earth* is the point of having refcounted
pointers which never have their reference count modified. It's a fair question!

There's three possible answers:

1: You don't need the refcounts anymore.

Hooray!

2: You have a mixed system. 

*Some* reference counted things can be aliased, and just want your runtime to be more uniform and simple. Reasonable!

3: You are using [the Swift COW trick](https://gankra.github.io/blah/swift-abi/#ownership). 

So whenever you want to "copy" a value type, you actually just copy the pointer and bump the reference count, and do a proper deep copy whenever a mutation occurs with reference count > 1. In this system, moves may be an important part of your ABI, because they don't modify the reference count. They can also just be a thing your compiler emits in certain places as an optimization.





## Ownership Strategy: Empty Sentinels (C++ RAII and Move Semantics)

Alright so this strategy of "empty values" is basically how C++ works, but the more I try to precisely describe how exactly C++ works, the more I'm going to put my foot in my mouth, so I'm gonna try to be a bit abstract here. The precise details of C++ might vary from what I describe.

If you don't want to have a uniform type layout, you can still implement an ownership system with runtime sentinels, but you need to be a bit more careful with it.

Basically, for types that we want to be able to conditionally "initialize" or "deinitialize" with moves, we need those types to define a sentinel value that basically means "I'm empty" and for their destructor to check for this value.

Where there's a bit of subtlety is exactly how "uninitialized" this value really is. For instance, if you have a type like `std::unique_ptr` which is just a wrapper around a heap allocation, then its "empty" sentinel value is just null again, and yeah you can definitely logically think of that as an "uninitialized" heap pointer. If you dereference it, Bad Things Will Happen, just like using any other kind of uninitialized memory.

But for something like say, an ArrayStack (`std::vector`), the empty sentinel can just be... a perfectly reasonable empty ArrayStack. Nothing bad will happen if you call `push_back` on it. But importantly (or at least *ideally*) an empty ArrayStack doesn't actually allocate anything.

So we can implement `drop` as something like:

```rust,ignore
struct ArrayStack {
    drop {
        if self.is_empty() {
            // nothing to do, noop
            return; 
        }

        for x in self {
            drop x
        }

        free(self.buffer)
    }

    // ...
}
```

With that implementation of `drop`, we now have the ability to implement C++-style moves. Now, C++ specifically lets the ArrayStack define all the fiddly details of how the `move` executes to support intrusive pointers, but we're not going to worry about that right now. We're just going to look at the high level idea any language could use.

Let's start with a super simple case, moving from one location to another:

```rust,ignore
let x = ArrayStack(...)
let y = move x
```

We can expand this to:

```text
%x = ArrayStack_init(...)  // Create a some ArrayStack
%y = %x                    // "move" it to y by just memcopying it
%x = ArrayStack_empty()    // Overwrite x with ArrayStack's "empty" value
drop y                     // drop y, will actually do work
drop x                     // drop x, will bail out early because it's empty
```

Simple, right? If we wanted to "really" do it like C++, it would look more like this:

```text
%x                         // x comes into scope (uninit)
ArrayStack_init(&x, ...)   // initialize x with some ArrayStack
%y                         // y comes into scope (uninit)
ArrayStack_empty(&y)       // default initialize y with an empty ArrayStack

ArrayStack_move(&x, &y)    // move the *contents* of x into y which will
                           // generally result in x being ArrayStack_empty

drop y                     // drop y, will actually do work
drop x                     // drop x, will bail our early because it's empty
```

The benefit of doing things this way is that if the address of `x` was significant (it contains pointers into itself, or some sort of global handler is holding a pointer to it), then it could have logic in its `move` implementation that fixes those pointers up when moving its *contents* over to `y`. Also the concept that an instance of a type lives in one location forever is maintained, which C++ likes.

Now let's forget about the nitty-gritty details of C++ and look at how this scales to a more complex example. Once again, we will be assuming you want basically no static analysis and want to be able to spew out your codegen as naively as possible.

```rust,ignore
let x

if cond1 { 
    x = ArrayStack(...)
}

if cond2 {
    consume(move x)
}
```

```text
%x = ArrayStack_empty()             // x default initialized to empty

if cond1 {
    %temp = ArrayStack_init(...)    // construct new ArrayStack in a temporary
    drop %x                         // destroy the old x (noop, empty)
    %x = %temp                      // bitcopy temp to x
    %temp = ArrayStack_empty()      // make temp empty
    drop %temp                      // drop temp (noop, empty)
}

if cond2 {
    %consume_args = ...;            // <some calling convention shit>

    %consume_args.arg1 = %x         // bitcopy x into the first slot
                                    // might be ArrayStack_empty, that's fine!

    %x = ArrayStack_empty()         // make x empty
    consume(consume_args)           // do the call
}

drop %x                             // drop x (may or may not be a noop)
```

This is basically the same thing we saw with with `null` sentinels, so similar arguments about optimizing out the `temp` cleanup apply.




## Ownership Strategy: Smuggled Drop Flags (Early Version of Rust)

> DISCLAIMER: some of this stuff predates my time working on Rust, so I wasn't there for this full history. I might be misremembering or misrepresenting some of the motivation and details here, but the high level concept is correct.

Rust doesn't have a uniform layout, so it can't rely on a sentinel like `null`. And it didn't want to have all the fiddly user-overloaded complexity of empty sentinels either. So early on they went for a very blunt but effective solution: have the compiler slap a "needs_drop" flag into every type that might need to be dropped.

In general, this meant every type with a destructor (or that contained a destructor) got a boolean slapped on the end of it, with all the type size bloat that entailed. Conceptually you can just think of this as every one of these types being implicitly wrapped in an `Option`. At that point the entire design looks *exactly* the same as we saw with null sentinels, but replace null with None. The generated code isn't quite as uniform, but it's basically the same.

There are some cute layout tricks you can apply, like using any bits that would otherwise be padding to store a drop flag, or using "forbidden values" as your drop flag (i.e. if a type is nonnull pointer, then the compiler can use null to mean None).

But this whole thing was very aesthetically... unappealing to Rust's target audience. For a long time it was generally understood that they were just a temporary hack that would be replaced by the "real" solution we'll see in the next section. It took a while, but indeed they were eventually replaced.

It's also worth noting that Rust did *have* Definite Initialization analysis, and it would generally statically reject programs that weren't statically sound. For instance, this wouldn't compile in Rust:

```rust,ignore
let x: MyType;

if cond1 { 
    x = MyType(1);
}

if cond2 {
    consume(x);     // ERROR: use of potentially uninitialized value
}
```

So why did Rust even *have* drop flags? It's because it specifically wanted to let calls to `drop` to be inserted dynamically by the compiler. As we have seen, it's very easy to sprinkle code with harmless noop destructor calls. Here's some examples where that's useful:


```rust,ignore
let mut my_string;

if cond1 {
    my_string = String::from("wow dynamic!")
    println!("{}", my_string);
}

// needs a dynamic drop!
my_string = String::from("maybe drops the old value?");  
println!("{}", my_string);
```

```rust,ignore
fn main() {
    // Neat trick that lets you "return" a reference to a value 
    // even though you're conditionally computing it in an inner scope.
    let temp;
    let my_borrowed_string;

    if true {
        temp = format!("computed string {}", "wow");
        my_borrowed_string = &*temp; // Ok! temp is init and outlives the scope
    } else {
        my_borrowed_string = "static string!";
    }

    // At this point we can't access `temp` because it's might be uninit
    // but `my_borrowed_string` is definitely init!
    println!("wow! {}", my_borrowed_string);

    // temp goes out scope here, needs to be dynamically dropped!
}
```




## Ownership Strategy: Stack Drop Flags (Rust and Swift)

So what was the visionary improvement upon Smuggled Drop Flags that Rust settled on?

Just put the drop flags on the stack like normal variables.

Yeah that's it.

When this was first proposed, it was called [non-zeroing dynamic drops](https://github.com/rust-lang/rfcs/pull/320). Nice work, pnkfelix!

This does come with limitations and complexity. The only smuggled implementation meant it was easier to think about dynamically initializing arrays or partially moving out of structs, because if you wanted to check if some value deeply buried in a sub-sub-sub-sub-field was initialized, the flag was just right there with it.

It's a lot more complicated to have to map that over to flags you have on the stack. I can't recall if this transition involved ripping out some really complicated cute tricks you could do, or if they managed to preserve all the old semantics. My gut tells me they broke like 1 or 2 really cute things that weren't a big deal.

Also once the flags were moved onto the stack, the compiler could be *much* more intelligent about only actually emitting and messing with them when the drops actually *were* dynamic. Most drops aren't dynamic, so this was a nice codegen cleanup.

Note however that removing non-dynamic drop flags means we have now moved into our first implementation strategy that actually *requires* some fairly sophisticated static analysis. To omit any drop flags, you need to have some form of Definite Initialization analysis that properly tracks the flow of values through the function. This means that when the compiler sees code like this, it can just not emit a drop at all:

```rust,ignore
let x = String::from("wow my owned value!!");
consume(x);
// x is now uninitialized, no drop emitted in the binary
```

(Of course this is all also complicated by the existence of unwinding, but literally everything is complicated by the existence of unwinding...)

I focused on Rust here for Story Telling reasons, but Swift uses this design as well.

It's not *terribly* surprising that they both converged on this solution, because as far as I know it's basically the "ideal" implementation of ownership if you're willing to put in the elbow grease, and both languages *love* putting in compiler elbow grease.

Of course, for Rust a lot of ownership is part of the surface semantics of the language -- if you mess up your moves it will be a compiler error. For Swift, this stuff shows up more as an implementation detail, than something a user thinks about day-to-day. Moves are a great optimization, so the compiler tries to find places in the program where it can insert moves instead of clones. Having dynamic drop flags makes it easier to move things more often.

[Although Swift is more seriously looking at exposing this stuff to the end-user to make the language more reliable for low-level stuff](https://forums.swift.org/t/a-roadmap-for-improving-swift-performance-predictability-arc-improvements-and-ownership-control/54206).

Ownership is a tool, and tools are for solving the problems you care about!



## Ownership Strategy: Static Drop (Rejected Rust Proposal)

Ah, now for a treat. 

Let us turn back the page in Rust's history. No, farther. FARTHER!

Ah, there it is.

Today children, we will be looking at a Rust That Could Have Been.

[Static Drop Semantics](https://github.com/rust-lang/rfcs/pull/210). (Oh hey, it's pnkfelix! sorry your proposal was rejected in favour of the one by... *checks notes*... also pnkfelix.)

So you might hear the phrase "static drop semantics", and picture "oh Rust was going to make it a compilation error to have any dynamic drops". Oh sweet child, no, this proposal was far more fantastic than that!

The proposal was to give the compiler latitude to *freely reorder destructors to make the drops static*. It's pure chaos and I *love it*.

To make it clear what that means, let's look at a little example:


```rust
struct Yells { 
    id: usize,
}
impl Drop for Yells {
    fn drop(&mut self) {
        println!("AAAAAAAA {}", id);
    }
}

fn main() {
    let var1: Yells;
    let var2 = Yells { id: 2 };

    if true {
        var1 = Yells { id: 1 };
    }

    println!("function over!");
}
```

This program has our variables print out a message when they're dropped. Variables go out of scope in the reverse order of their declaration, so this prints:

```text
function over!
AAAAAAAA 2
AAAAAAAA 1
```

Because the compiler desugars the program to this:

```rust,ignore
    let var1: Yells;
    let var2 = Yells { id: 2 };

    if true {
        var1 = Yells { id: 1 };
    }

    println!("function over!");

    drop(var2);
    drop(var1);
```

But with Static Drop Semantics, the program would instead emit this:

```text
AAAAAAAA 1
function over!
AAAAAAAA 2
```

Because the compiler would desugar the program to this:

```rust,ignore
    let var1: Yells;
    let var2 = Yells { id: 2 };

    if true {
        var1 = Yells { id: 1 };
        drop(var1)
    }

    println!("function over!");
    drop(var2)
```

Tragically this proposal was just a *bit* too wild for Rust's developers at the time, so we never got to really experience this. *Curse you pnkfelix, why couldn't you take pnkfelix's revolutionary ideas seriously!!*



# IM FUCKING DONE

AAAAAA THIS ARTICLE WAS SUPPOSED TO BE A BULLETED LIST HOW DID THIS HAPPEN

WHY DO I KEEP DOING THIS

IS THIS EVEN COHERENT

I DON'T EVEN KNOW

IM NOT ABOUT TO REVIEW AND EDIT THIS 37 PAGE THESIS(???) ON UNINITIALIZED MEMORY THAT I MADE IN FEVER DREAM THIS WEEKEND

PLEASE FREE ME
