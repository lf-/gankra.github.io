% Rust VS Swift 

Alexis Beingessner

[<img src="swiftsvg.png" width="250" style="display:inline; box-shadow:none;"></img>]
(https://swift.org)
[<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://cglab.ca/~abeinges)
[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>]
(https://rust-lang.org)



<!--
Hey There, I'm Alexis! 

I've spent the last three years working on Swift and Rust; but mostly Rust.

I find putting the two of them side-by-side fascinating. Partly for the similarities, but moreso for the differences.

Let's take a look at the similarities first:
-->



# Swift is...

* a new open source systems language
* developed by a major browser vendor
* to replace The Son Of C
* and make programming easier




# Featuring:

* value types! 
* no garbage collection!
* interfaces over inheritance!
* tagged unions and pattern matching!
* closures!
* easy compatibility with legacy C codebases!



# Oops

That's Rust!

<!--
there are a lot of parallels you can draw between the two languages, especially
if you look at them "from space".
-->



# BFFs 4 Eva

Swift: hey check out `if let`

Rust: imma steal that (but make it better)

Swift: imma steal that back

<!--
The two have even developed features cooperatively.

The most notable example is if-let:

* Swift created if-let as an easy way to work with optional values

* Rust liked it, but generalized it to work with any pattern

* Swift then came back around and copied that ability back into its own design
-->




# But Really They're Quite Different

(Disclaimer: IMHO)

Rust rocks way harder at systems programming

Swift rocks way harder at app development

<!--
Systems programming: kernels, drivers, databases, allocators, parsers, renderers

Application programming: GUIs, localization, HTTP, processing data sets
-->



# Why Is That?

different motivations and users -> different philosophies

<!--
So why is it that the two are better at different things?
-->


# Motivation: Rust

"Safe, Fast, Concurrent"

Firefox! 


<!-- 
Rust was borne of an exhaustion with C++ in Firefox. 

C++ is the source of most of the major exploits in Firefox.

At the same time: C++ is holding back the performance of Firefox. It's way
too hard to write *correct* concurrent or optimized C++ code.
-->



# Motivation: Swift

"Safe, Fast, Expressive"

iOS Apps!

<!--
Swift was borne of an exhaustion with Objective-C in Apple's ecosystem.

Objective-C has many of the same problems as C++ (hence the same focus on
"safe and fast") but the environment was different. 

Now, the Swift devs are very clear that they intend Swift to be use for everything, from low level systems to little scripts, but if you look at what people are using Swift for, it's very clear that Swift's biggest influence is iOS apps.

Caring about things like memory management
in the guts of a web browser is expected. It's harder to make that argument for
a simple app, especially when it might be the author's first ever programming experience.

Sure, that stuff should be there when your app gets more complex and it starts to be an issue, but it shouldn't be something you need to worry about on day 1.

Rust on the other hand, with its Firefox-like perspective, is assuming you want to care about these things on day 1.
-->





# Philosophies - Swift VS Rust

* Progressive Complexity VS Essential Complexity
* Conveniences VS Uniformity
* Semantics VS Implementation

<!--
From these motivations, Swift and Rust developed divergent core philosophies.

Now to be clear: these aren't hard rules, but tendencies. You can find places
where both languages have gone either way when trying to resolve an issue.
But where the two have *diverged*, you can often find these differences. If you talk to the developers of Swift and Rust, they'll even point out places where they think they maybe overdid it a bit, and want to be more like the other language.

But languages have momentum, so these tendencies will probably live forever.

Let's check these philosophies out.
-->



# Swift: Progressive Disclosure of Complexity

There should be a simple ways to do things correctly

complexities are hidden ...but available when you care

<!--
PDoC serves two purposes: 
* for people new to a problem or library, it gives a gentle learning curve
* for people used to a problem or library, it lets them ignore the specific details that don't matter to them
-->




# Features For Progressive Disclosure: Functions

* labeled args
* variable length args
* default args



# Ex: Print

```
print(name)
```

# Ex: Print (varargs)

```
print(name1, name2, name3)
```

# Ex: Print (labels/defaults)

```
print(name1, name2, name3, 
      seperator: ";")
```

# Ex: Print (labels/defaults)

```
print(name1, name2, name3, 
      seperator: ";", 
      toStream: outputFile)
```


<!--
Rust will probably have some of these features one day...

but they just aren't considered that important, since PDoC isn't one of its core philosophies. In fact, there are members of the community that oppose these features precisely *because* they hide details.

That's because Rust has the philosophy of...
-->




# Rust: Essential Complexity

some complexity is essential, and progress must be halted for it

it's ok -- the compiler will help you learn

<!--
For new people, this can represent a "learning wall" that must be overcome. But hopefully once it's overcome, they've actually learned some of the complexities of the domain, and are a better programmer.

For experienced people, this generally means there's less "gotchas" in the language. If it compiles, it works!
-->



# Features for Essential Complexity: Ownership

* moves
* borrows

<!--
Ownership is an *infamous* case of the essential complexity philosophy. The
community even has a term for this: "fighting the borrow checker". 

The particular complexity this is primarily targetted at is memory management.
-->



# Ex: Use After Move

```
let array = vec![1, 2, 3, 4];
process(array); // seems
process(array); // legit
```

<!--
Pretty simple code: make an arraystack, and pass it to two functions
-->



# Ex: Use After Free (Error)

```text
error[E0382]: use of moved value: `array`
  |
3 |     process(array); // seems
  |             ----- value moved here
4 |     process(array); // legit
  |             ^^^^^ value used here after move
  |
  = note: move occurs because `array` ... does not implement the `Copy` trait
```

[E0382](https://doc.rust-lang.org/error-index.html#E0382)

<!--
...and we get a compilation error.

(read it)

So if you've used Rust for like a week you'll probably have run into this and
understand it. Hopefully what to do about it also comes naturally, (in this case, probably a copy-paste error?)

But if you're new to Rust, it's totally reasonable to feel walled... what is this? But that's what the error code is for: if we look it up [click], we get a great pile of documentation written about why this error exists, what might
have caused it, and what you can do about it.



Swift has been working towards an ownership model inspired by Rust, but Rust's
design was considered too much to put on programmers up front; it's inconsistent with the goal of progressive disclosure. 

Swift's version of Ownership will be more of an "opt in" design for optimizations.
-->




# Rust: Uniformity

Everything is...

* an expression
* a value
* a pattern
* a trait
* a function

<!--
It's really common in Rust to hear "we don't have that because everything is (read list)".

This gives Rust code a certain uniformity and shallowness. There's like 5 or 6
concepts you really need to know, and then everything else is idioms.
-->



# Ex: Constructors (Everything Is A Fn)

```
impl Foo {
    fn new() -> Foo { Foo { cap: 1 } }
    fn with_capacity(cap: usize) -> Foo { Foo { cap: cap } }
}

let x = Foo::new();
let y = Foo::with_capacity(10);
```

<!--
A lot of languages have the concept of a type constructor. Rust doesn't bother
to provide that as a first-class notion. Instead, *everything is just a function*.

So you don't need to learn about constructors, but you do get a bit of a  "broken record" effect, as code can repeat itself a lot.
-->



# Ex: Ternary (Everything Is An Expr)

```
let val = if condition { x } else { y };
```

<!--
It's pretty common for people to show up to Rust and ask "how do I do a ternary", which gets them the "everything's an expression" spiel.
-->





# Swift: Conveniences

common pattern? make it a feature!

constructors? `Check()`

ternary? `check ? check : check`


<!--
By contrast, Swift has tons of special forms for common patterns.
-->


# Ex: Swift Constructors

```swift
extension Foo {
    init() { self.cap = 1 }
    init(capacity: Int) { self.cap = capacity }
}

let x = Foo()
let y = Foo(capacity: 10)
```

<!--
Here we see that Swift's version of constructors is a lot more concise, and lacks any redundant information.

However these more concise forms tend to come at a cost. Whenever you want
to do something that doesn't fit into the common pattern it's trying to
match, things tend to get a bit awkward:
-->



# Ex: Swift Constructors (Limits)

```swift
extension Foo {
    init() { ... }
    init(secure: ()) { ... }             // ???
    static func secure() -> Foo { ... }  // ???
}

let x = Foo()
let y = Foo(secure: ())
let z = Foo.secure()
```

<!--
Here we want provide two constructors which take the same arguments but have
different behaviour. Swift's design for constructors can't handle this case
cleanly, so we have to hack around it.
-->


# Ex: Rust Constructors (No Problem)

```
impl Foo {
    fn new() -> Foo { ... }
    fn secure() -> Foo { ... }
}

let x = Foo::new();
let y = Foo::secure();
```

<!--
Here Rust's philosophy of uniformity means that we still have function names
to disambiguate while remaining idiomatic.
-->




# Implementation VS Semantics

Should the user care...

* about allocations?
* about costs?
* about tricky optimizations?



# Rust: Implementation

```
let mut x = vec![1, 2, 3, 4];
let y = x.clone();  // explicit sheep copy
x[0] = 5            // what you expect
```

<!--
Rust tries to focus on implementation, and wants you to be able to see
potentially expensive things.

Here we create an arraystack...

`clone` here explicitly makes a new allocation and copies the contents of the array.

The assignment on the next line doesn't do anything interesting. It just sets the value.
-->


# Swift: Semantics

```swift
var x = [1, 2, 3, 4]
let y = x            // ~magic~
x[0] = 5             // 🐮
```

<!--
Swift on the other hand is focused on semantics. This code does the same thing
as the Rust code, but there's several hidden details.

Arrays in swift are actually Copy-on-Write, so the assignment here actually
increments a reference count, and y and x will both point to the same memory.

*Semantically* x and y are distinct values, but in their implementations, they're actually the same one!

During the assignment `x` detects that it's aliased by `y`, and creates a fresh
allocation for itself. So a seemingly boring operation like setting a value in an array can trigger a very expensive operation.
-->



# Implications of Implementations

* easy to *understand* code
* easy to do *tricky* things

-----

* developers overreact and freak out
* loses optimization opportunities




# Situation with Semantics

* sloppy code can go faster
* better learning curve

-----

* optimizations become necessary, performance unpredictable
* hard to do tricky things






# Philosophies - Swift VS Rust

* Progressive Complexity VS Essential Complexity
* Conveniences VS Uniformity
* Semantics VS Implementation
