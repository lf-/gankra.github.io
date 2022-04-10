% Defaults Affect Inference in Rust: Expressions Instead Of Types

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">April 10th, 2022</p>
</header>

Have you ever noticed that some of the bounds on Rust's collection types are weird? This is a very long and old story of a problem from before even 1.0. I will explain this problem, and then try to solve it in a new way. I will fail to do so, but in the process develop out a lot of interesting concepts, and maybe someone a little smarter than me can find the "missing piece" to perfectly solve the problem.


# Default Type Parameters

```rust
pub struct HashMap<K, V, S = RandomState>;

pub fn new() -> HashMap<K, V, RandomState>;
pub fn default() -> HashMap<K, V, S>;
```

[HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) has a third type parameter that you may have not noticed, declared as `S = RandomState`. S is the [Hasher](https://doc.rust-lang.org/std/hash/trait.BuildHasher.html) HashMap uses, and we're saying that if it isn't otherwise specified, to implicitly use [RandomState](https://doc.rust-lang.org/std/collections/hash_map/struct.RandomState.html), which protects users against accidentally being vulnerable to [Hash DOS Attacks](https://gankra.github.io/blah/robinhood-part-1/#hashing). This is a safe default, but not always necessary, so users can explicitly specify their own S to make seeding deterministic, or just to make the hashing algorithm weaker (and presumably faster).

But why does `new` hardcode RandomState, while `default` doesn't?

Well, it's because defaulted type parameters are... not as good as you would hope. If you use HashMap as a type, as in `my_map: HashMap<K, V>` then rust will implicitly understand that this is actually just shorthand for `HashMap<K, V, RandomState>`. But if you refer to HashMap as part of an expression, as in `HashMap::some_static_method` then the default isn't "turned on" -- all 3 of K, V, and S are free to be inferred from context.

Rust is pretty all or nothing with type parameters: either you're specifying that something *has* some type, all the generics need to be filled in, and the default is applied; or you're using the type name as a glorified namespace for its methods, all of the generics are freely inferred, and defaults aren't applied. (This line gets blurrier when you write `let x: HashMap<_, V> = ...` but don't worry about it.)

Inferring the types in the "expression" context is very handy, because it lets you not write any type parameters when they can be obviously inferred from usage. For instance, this works:

```rust
let mut my_map = HashMap::new();
my_map.insert("hello", "there");
println!("{:?}", my_map);
```

But this doesn't:

```rust
let mut my_map = HashMap::new();
println!("{:?}", my_map);
```

```text
error[E0282]: type annotations needed for `HashMap<K, V>`
 --> src/main.rs:4:22
  |
4 |     let mut my_map = HashMap::new();
  |         ----------   ^^^^^^^^^^^^ cannot infer type for type parameter `K`
  |         |
  |         consider giving `my_map` the explicit type `HashMap<K, V>`, where the type parameter `K` is specified
```

The problem is that we "need" to know what type the HashMap has, but we're only using operations that are completely generic so... it could have any type! We can help rustc out with a type ascription:

```
let mut my_map: HashMap<String, u32> = HashMap::new();
println!("{:?}", my_map);
```

or the turbofish:

```rust
let mut my_map = HashMap::<String, u32>::new();
println!("{:?}", my_map);
```

But hold on, how is the compiler inferring S in this snippet?

```rust
let mut my_map = HashMap::new();
my_map.insert("hello", "there");
println!("{:?}", my_map);
```

What possibly determines it? Well, we can find out the hard way by using HashMap::default instead:

```rust
let mut my_map = HashMap::default();
my_map.insert("hello", "there");
println!("{:?}", my_map);
```

```text
error[E0282]: type annotations needed for `HashMap<&str, &str, S>`
 --> src/main.rs:4:22
  |
4 |     let mut my_map = HashMap::default();
  |         ----------   ^^^^^^^ cannot infer type for type parameter `S` declared on the struct `HashMap`
  |         |
  |         consider giving `my_map` the explicit type `HashMap<_, _, S>`, where the type parameter `S` is specified
```

YUP! HashMap::new hardcoding `S = RandomState` is *load bearing*. Any code that creates a short-lived HashMap as part of some algorithm would run into this horrible error about how S is unknown! The consequence (suffering) of this hack is that anyone who *does* want to change S just... doesn't get to use `new`! This is why it's common to see:

```rust
type MyMap<K, V> = HashMap<K, V, MyHasher>;

let map = MyMap::default();
...
```

We were sneaky and cut out a little "hole" in the API where it's generic, so that you still have Basically Nice Things. Unfortunately this is a hack that's *pretty* unique to HashMap because its default type parameter was *ancient*. Like, pre-1.0 ancient. In terms of API stability, HashMap has *always* had S.




# Adding New Defaulted Parameters

This is not the case for [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html), which many years after 1.0 got a new type parameter grafted onto it. Of course that would be a breaking change since lots of code refers to `Vec<T>`, so we of course reached for the default type parameter machinery: 

```rust
pub struct Vec<T, A = Global> where
    A: Allocator, 
```


Ah, the mythical generic [Allocator](https://doc.rust-lang.org/std/alloc/trait.Allocator.html), a feature long heralded since before even 1.0. They're finally [experimenting with it](https://github.com/rust-lang/rust/issues/32838). This is a feature that, during my tenure as a standard library team member, I basically soft-blocked on "solving" the HashMap hackery. Why? Let's look at Vec's constructors:

```rust
// Old, stable
pub fn new() -> Vec<T, Global>;
pub fn with_capacity(capacity: usize) -> Vec<T, Global>;
pub fn default() -> Vec<T, Global>;
pub unsafe fn from_raw_parts(ptr: *mut T, length: usize, capacity: usize) -> Vec<T, Global>;

// New, unstable
pub fn new_in(alloc: A) -> Vec<T, A>;
pub fn with_capacity_in(capacity: usize, alloc: A) -> Vec<T, A>;
pub unsafe fn from_raw_parts_in(/* you get the idea */) -> Vec<T, A>;
```

Yiiiikes. All the "good" constructors hardcode the allocator to the "classic" Global allocator, and you need to use different constructors if you want a different allocator (even if it's globally accessible too!). Worse yet, `Vec<T, MyAllocator>` *can't implement the [Default](https://doc.rust-lang.org/std/default/trait.Default.html) trait*.

We always knew this "had" to happen with custom allocators, because if you were to make `Default` generic over `A` then *all code that calls Vec::default would become instantly ambiguous*. Making a ton of code stop compiling is obviously terrible, so, we didn't! I considered this "fatal" to custom allocators, but people Really Really Want Them, so I totally understand pushing through the pain anyway.

I was also willing to consider it "fatal" because... well, there was supposed to be a solution!




# Defaults Affect Inference

Lets go all the way back to February 2015, 3 months before Rust 1.0, where we had all agreed on the solution to this issue: [RFC 213: Default Type Parameter Fallback](https://github.com/rust-lang/rfcs/blob/master/text/0213-defaulted-type-params.md), or as I've always called it, Defaults Affect Inference (DAI).

The premise is simple: if the compiler is Doing Some Inference and can't infer what some generic type parameter is, but there's a Default on the type... use it!

The precedent is simple: the compiler *already does this* for integer literals! If absolutely nothing at all implies what type an integer literal should have, [it just picks i32](https://github.com/rust-lang/rfcs/blob/master/text/0212-restore-int-fallback.md) to make the code work. If this sounds Completely Fucking Terrifying... it's actually pretty rare in practice. In "real" code there's almost always something like array indexing or your inputs/outputs that implies everything should have some particular type (Rust doesn't do integer promotion and doesn't like mixed-with operations). This feature is basically "make unit tests and code examples less annoying". (Don't ask about why it's i32.)

So! Slamdunk feature, makes perfect sense, compiler already does this trick. Pretty natural for me, a library person, to say "yep that's happening, wait until it's shipped".

Well, as it turns out, [it's a more complicated feature than it sounded](https://github.com/rust-lang/rust/issues/27336). First there was some concern over how exactly it should [interact with i32 fallback](https://internals.rust-lang.org/t/interaction-of-user-defined-and-integral-fallbacks-with-inference/2496) but that one seemed... fairly negligible. No one wanted to default integer types for Hashers or Allocators, and those were the important things! But as time went on the compiler folks found more [hairy questions](https://github.com/rust-lang/rust/issues/27336#issuecomment-229988042) and things just got too nasty and... well the feature fell into limbo. In 2020 everyone just said let's stop pretending we're gonna crack this and [the tracking issue for the RFC was simply closed](https://github.com/rust-lang/rust/issues/27336#issuecomment-662951104).

I'll be honest, I don't really "get" all the problems. It seems to be a mix of genuinely tricky corner cases mixed with fundamental limitations in the current inference engine (which has been descripted to me as "basically [bidirectional typechecking](https://ncatlab.org/nlab/show/bidirectional+typechecking) but with some hacky subtyping-plus-[Hindley-Miln](http://dev.stephendiehl.com/fun/006_hindley_milner.html)-ish thing grafted on").

But I have been on a kick of solving problems that I can't understand with [solutions that are obviously correct](https://gankra.github.io/blah/tower-of-weakenings/), so I don't *need* to understand the problem if I can make something simpler that seemingly "has to work". Let's try!



# Some Wild Shit Swift Does

So if you ever talk to me about programming language design, this conversation will happen a lot:

You: "Wouldn't it be cool if a language had \<wild feature\>"
Me: "[Oh yeah Swift did that.](https://gankra.github.io/blah/swift-abi/)"

As far as I can tell, Swift is basically what happens when you hypnotize a bunch of compiler developers to simply believe that what ever you say "should" work *does* work, and then they start walking through walls until it *does*.

There's also a very long tradition of both Rust and Swift just blatantly pointing at eachother and just going "nice, that's mine now":

* [Rust copies if-let from Swift](https://github.com/rust-lang/rfcs/blob/master/text/0160-if-let.md)
* [Rust copies guard-let from Swift](https://github.com/rust-lang/rfcs/blob/master/text/2294-if-let-guard.md)
* [Swift copies "impl Trait" types as "some Protocol"](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)
* [Swift proceeds to make "some Protocol" more like Rust](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)

This rules! Languages should absolutely just do this whenever another language clearly nails a problem they're working on. So, if you're working on Swift and want some inspiration, scroll through [Rust's RFCs]() and if you're working on Rust and want some inspiration, scroll through [Swift Evolution](https://github.com/apple/swift-evolution/tree/main/proposals).

I was doing that last night and I came across this Swift Evolution proposal, [SE-0347: Type inference form default expressions](https://github.com/apple/swift-evolution/blob/main/proposals/0347-type-inference-from-default-exprs.md). Sounds pretty fuckin' relevant to my interests!

In this proposal, they're looking at a problem that's like, 3 steps deeper into rad features than Rust is right now. 

* Swift has named function arguments (not used in the following example, but useful later)
* Swift has optional arguments with defaulted values
* Swift has overrideable literals (not needed by us, but Fun To Mention)

Putting these things together with generics, you can in principle write something like this totally artifical example:

```
func compute<C: Collection>(_ values: C = [0, 1, 2])
```

This code is declaring a `compute(my_values)` function that works on any Collection, but if you don't want to provide the data to operate on it, you can just call `compute` and it will use the default value. But the default value is an array literal, so any Collection that *also* conforms to [ArrayLiteralConvertible](https://swiftdoc.org/v3.0/protocol/arrayliteralconvertible/) can use this. Also [the integer literals are overloadable](https://swiftdoc.org/v3.0/protocol/integerliteralconvertible/) so this is [actually a combinatoric type inferrence nightmare](https://stackoverflow.com/questions/29707622/swift-compiler-error-expression-too-complex-on-a-string-concatenation), but that's just how Swift rolls.

The crux of the issue is that not all Collections conform to ArrayLiteralConvertible, so this default isn't universally applicable. The compiler understandably and conservatively says "no" to this, but, that was only the a brief weakening of the Swift team's hypnotic trance, so this proposal just says "actually yes". The interpretation of this is conceptually simple: if you don't provide this argument, then it's "as if" you explicitly wrote out the default expression, and the compiler infers the code like normal from there.

They demonstrate this with a "real" example of:

```
struct Box<F: Flags> {
  init(flags: F = DefaultFlags()) {
    ...
  }
}

Box() // F is inferred to be DefaultFlags
Box(flags: CustomFlags()) // F is inferred to be CustomFlags
```

Hey! That's friggin' HashMap's Hasher! This is doing Defaults Affect Inference!

But crucially, the defaulting is *not* keying off the type, and is instead keying of the *method*. Why is this crucial? Because the case where we *are* doing Default Affect Inference, it's literally just sugar for:

```
Box(flags: DefaultFlags()) 
```

I think we can all agree that the compiler *definitely* already knows how to infer that, because it's literally what we ask you to do with [Vec::new_in](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.new_in) and [HashMap::with_hasher](https://doc.rust-lang.org/std/collections/hash_map/struct.HashMap.html#method.with_hasher):

```rust
pub fn new_in(alloc: A) -> Vec<T, A>;
pub fn with_hasher(hash_builder: S) -> HashMap<K, V, S>
```

So if our solution is *literally* a desugarring to extra optional arguments, then It Definitely Just Works!



# Rust Features You Want Anyway

Ok so, here's the features Rust should steal from Swift that people want anyway:

* Named arguments
* Optional arguments with default expressions

Optional arguments are the only feature we "really" need, but named arguments "soft block" them because once you open the optional argument floodgates, callsites get really complicated and confusing without them! It also makes it unambiguously possible to have multiple optional arguments and elide any subset of them. For this reason I would make optional arguments *have* to be named, because it's better for forward-compatibility (you can always add more optional defaults to the end and it's just as easy to only use the new ones and ignore the old ones).


## Named Arguments

In Swift, named arguments are declared like this:

```swift
func ditto(arg: String) { print(arg) }
func external(withArg arg: String) { print(arg) }
func unnamed(_ arg: String) { print(arg) } 
```

and must be used like this:

```swift
ditto(arg: "hello")
external(withArg: "there")
unnamed("everyone!")
```

To summarize: arguments are named by default, and function calls must use that name (it's even part of the function signature!). **The caller cannot change the order of the named arguments.** If you put *two* names, then the first is "external" and the second is "internal". If you put `_` as the external name, then it's a purely positional argument.

These rules are very interesting because they're... not what you expect coming from a language like Python where (AIUI) named arguments are basically sugar for an unordered Dictionary as far as the caller is concerned.

Swift is using named arguments to make the callsite *clear* and *natural to read*. So for instance this array search API:

```swift
func starts<PossiblePrefix>(
    with possiblePrefix: PossiblePrefix, 
    by areEquivalent: (Element, PossiblePrefix.Element) throws -> Bool
) rethrows -> Bool 
where PossiblePrefix : Sequence
```

is called as:

```swift
array.starts(with: ["x", "y"], by: my_comparator)
```

It is extremely cute but also extremely unironically *nice*.

How would we "steal" this for Rust?

Well we can't have the same default for backcompat, but external args are pretty common when you want to do this, so let's take a simple approach:

```swift
func ditto(arg arg: String) { println!("{arg}") }
func external(withArg arg: String) { println!("{arg}") }
func unnamed(arg: String) { println!("{arg}") }

ditto(arg: "hello")
external(withArg: "there")
unnamed("everyone!")
```

Basically, you just always have to use external args. If you want you could make `ditto` be `_ arg`, you can, but is an amazing nightmare because it would mean Rust and Swift have the exact same syntax and support the exact same things *but the mapping from syntax to behaviour is perfectly shuffled and never the same*. This is pure chaos and I respect it entirely.

Proponents of "Rust should totally have arbitrary subexpression type ascription" will be sad that I'm "burning" the `:` here but:

* It's 2022 and it still hasn't happened, get over it
* This is already rust's syntax for initializing records, so it's just Consistent.
* Named arguments are way higher impact than defaults

I am perfectly happy to accept all the following restrictions Swift applies, but I concede that they're fair game for bikeshedding because we are adding this stuff to an existing language:

1. Named arguments cannot be reordered by the caller
2. Named arguments must be passed using the name (can't be positional too)
3. Argument names are part of the function signature (for function pointers and the like)

Restriction 1 is just Opinionated but does help ensure a "flow" to the args, and is mildly necessary to make function signatures coherent. Restriction 2 is a bit messy because it basically means no one gets to "phase in" named arguments to existing code. I expect this one will get dropped, but I think it's back-compat to "undo". Restriction 3 is similar to 2, although might get dropped even more aggressively because it would fuck with function type syntax (Swift lets you give tuples named fields so this is slightly easier to rationalize but that in itself is a huge tarpit that led to tons of wild problems).



## Optional Arguments

Ok now the "key" part. Swift lets you declare Optional Arguments with `=`:


```swift
func doThing(withArg arg: String = "hello")

// sugar for doThing(withArg: "hello")
doThing() 
```

I am *certain* this is wildly complicated in Swift because of all its wild features, but one simple interpretation of this is that this is equivalent to:

```
func doThing_argDefault() -> String { return "hello"; }
func doThing(arg: String);

doThing(arg: argDefault())
```

That's it! You just need to generate an anonymous free function that evaluates the expression and returns the field type. What would this look like in Rust? Let's try this:

```rust
fn do_thing(with_arg arg: &str = "hello")

// sugar for do_thing(with_arg: "hello")
do_thing()
```

Full desugarring, we get:

```rust
fn do_thing__with_arg_default() -> &'static str { "hello" }
fn do_thing<'a>(with_arg arg: &'a str);

do_thing(with_arg: do_thing__with_arg_default())
```

I used `&str` here on purpose because lifetimes are a bit of a nasty question, but I think with a hopefully simple answer: the return type of the default argument function is exactly the type of the argument *but with every lifetime replaced with 'static*. This requires the expression to be computable with no context, and because almost everything is covariant, will generally Do The Right Thing.

It will fall over for weird things involving invariance or contravariance and you will just get a compiler error. I think that's simply *fine*. The overwhelming application of this feature will be for things that don't even have lifetimes, like RandomState or Global. String literals and slices are the "big" issue, and 'static works well for those. `self` cannot be used to compute the default, because, oh my god no.

Also, if you're worried about type declaration having arbitrary expressions, we can go for the "salty but simple" solution of forcing the user to write the default function manually and provide a function pointer:

```rust
fn my_arg_default() -> &'static str { "hello" }
fn do_thing(with_arg arg: &str = my_arg_default)

// sugar for do_thing(with_arg: my_arg_default())
do_thing()
```

That makes the entire syntax as boring as it can get, and basically answers all the hard syntactical problems with "well it's all normal rust code you write". It's just a question of if the minimal syntax is *at all* viable. And the minimal syntax is purely a bikesheddable detail, so, presumably there exists *some* valid syntax for this!

The nastiest question as far as I'm concerned is *what is the actual signature of do_thing*. Ignoring "names in singatures", is it `fn() -> ()` or `for<'a> fn(&'a str) -> ()`? I would like to propose that the answer can be *both*. In particular, you should be allowed to do this:


```rust
fn my_arg_default() -> &'static str { "hello" }
fn do_thing(with_arg arg: &str = my_arg_default)

let my_func1: fn() -> () = do_thing;
let my_func2: fn(&'static str) -> () = do_thing;
```

Basically, whenever the compiler needs to coerce a function to a function pointer, it now has options to choose from, but in general it should be able to know "which one" to pick based on the coercion target, as is the case for many coercions. This is 100% me bullshitting, but it sure sounds plausible!

But, how *can* it do the `my_func1` coercion? Well, all the default calls are free functions with no inputs that return `'static`, so the compiler should *in principle* be able to emit this:

```rust
fn _do_thing_defaulted() {
    let arg0 = my_arg_default();
    do_thing(with_arg: arg0)
}

let my_func1: fn() -> () = _do_thing_defaulted;
```

This is an *extremely* boring magical transformation, because it's basically just:

```rust
let my_func1: fn() -> () = || -> () {
    let arg0 = my_arg_default();
    do_thing(with_arg: arg0)  
};
```

Which I believe actually works and compiles today.

(There's deeper corner of this for generics, multiple defaulted args, and traits... we'll be getting to that slowly as we refine the design.)





# Solving Hashers and Allocators: Fantasy Version

(I'm about to skip over some steps to show "the dream". This is intentionally making mistakes because it's useful for exposition to make the mistakes and then "solve them".)

Ok so how do we put this to service in solving Defaults Affect Inference? Well, we just add new named default args to all the old constructors! (Here I'll be using the sugariest version of the syntax, but know that I could pull it all out to the super desugarred forms)


```rust
impl<K, V, S: BuildHasher> HashMap<K, V, S> {
    fn new(with_hasher hasher: S = RandomState::new())
}

impl<T, A: Allocator> Vec<T, A> {
    // Yes, Global is an empty struct so this is an actual expr
    fn new(in_alloc allocator: A = Global)
}
```

And now you can do this:

```rust
let vec1 = Vec::new();
let vec2 = Vec::<MyAllocator>::new();
let vec3 = Vec::new(in_alloc: &my_local_allocator);

let map1 = HashMap::new();
let map2 = HashMap::<MyHasher>::new();
let map3 = HashMap::new(with_hasher: MySeededHasher::new(0));
```

Wow! Perfect! Incredible! Complete Bullshit That Doesn't Work!

I've skipped two things:

* I didn't introduce the feature the Swift proposal was *actually* about, which was allowing optional argument expressions to not cover all possible types for the argument.
* I made the defaults hardcode the default type parameter, but then just magically assumed `Vec::<MyAllocator>::new()` would work, which is nonsense.

We're going to need to think a bit harder to make this work!




# Subset Defaults

Going back to super-desugarred form, Subset Defaults would allow this code to compile:

```rust
fn default_hasher() -> RandomState { ... }

impl <K, V, S: BuildHasher> HashMap<K, V, S> {
    fn new(with_hasher hasher: S = default_hasher) { ... }
}

// uses S=RandomState
let map1 = HashMap::new();
let map2 = HashMap::new(with_hasher: MySeededHasher::new(0));
```

Under the hood, what this is actually desugarring to is something roughly like this:

```rust
fn default_hasher() -> RandomState { ... }

impl <K, V, S: BuildHasher> HashMap<K, V, S> {
    fn new(with_hasher hasher: S) { ... }
}


// uses S=RandomState
let map1 = || {
    let arg0 = default_hasher();
    HashMap::new(with_hasher: arg0);
};

let map2 = HashMap::new(with_hasher: MySeededHasher::new(0));
```

Basically the "canonical" signature has the compiler strip away all the defaults and just include all the names, but the compiler records the defaulting functions associated with each function. When *syntactically* you omit an argument (or a coercion site's type requires it), the compiler will look up the appropriate defaulting functions and emit a "reabstraction thunk" (closure). That calls the defaults and passes them in.

Under this desugarring, default subsets "obviously" work, because we're applying a transform *purely* based on syntax to "normal" rust code and then typechecking it. Good compiler errors may be a challenge, or maybe it will be fine, not sure!

This handles the "hardcode" usecase, but what about the "default" usecase. We could instead write this:

```rust
// NOTE: now generic over Default!
fn default_hasher<S: Default + BuildHasher>() -> S { S::default() }

impl <K, V, S: BuildHasher> HashMap<K, V, S> {
    fn new(with_hasher hasher: S = default_hasher) { ... }
}

let map = HashMap::<MyHasher>::new();
```

Now our `new` impl has a defaulting value implementation that is generic, but a more refined generic than `new`'s own signature. The desugarring would actually be completely unaffected:

```rust
fn default_hasher<S: Default + BuildHasher>() -> S { S::default() }

impl <K, V, S: BuildHasher> HashMap<K, V, S> {
    fn new(with_hasher hasher: S) { ... }
}


// uses S=RandomState
let map = || {
    let arg0 = default_hasher();
    HashMap::<MyHasher>::new(with_hasher: arg0);
};
```

Again, we have made a purely syntactic transform. Again, good compiler errors for the case where MyHasher doesn't implement Default is Probably Hard.

Now it would be good for the compiler to check if our defaults *at all* make sense. For instance, these two examples should ideally be rejected without any callers:

```rust
// Bound isn't a subset, doesn't include BuildHasher 
fn default_hasher<S: Default>() -> S { S::default() }
fn new<S: BuildHasher>(with_hasher hasher: S = default_hasher)
```

```rust
// Concrete type doesn't implement BuildHasher
fn default_hasher() -> String { String::new() }
fn new<S: BuildHasher>(with_hasher hasher: S = default_hasher)
```

I'm a bit tired so I'm not sure how to express this well, but I believe the compiler should be able to check this using normal type checking machinery. In particular, validating defaults is effectively just checking that the real constructor *could be called from the defaulting function*. So it needs to check that this code is well-formed:

```rust
// fake_default_hasher has the same bounds as default_hasher 
fn fake_default_hasher<S: Default>() { 
    let test = default_hasher()
    
    // Extra arguments added to demonstrate that we can do the everybody_loops
    // trick to potentially check things piece-wise.
    HashMap::new(
        arg0: todo!(), 
        with_hasher: test, 
        arg2: todo!()
    )
}
```

I... *think* default checking can be done piece-wise? And I *think* this should be "easy" for the compiler to either synthesize or check-without-even-generating-the-definiton? BIG HANDWAVE ON THIS ONE!





# Solving Hashers and Allocators: Reality Version

I don't think I can give you everything. The Dream was this:

```rust
let vec1 = Vec::new();
let vec2 = Vec::<MyAllocator>::new();
let vec3 = Vec::new(in_alloc: &my_local_allocator);

let map1 = HashMap::new();
let map2 = HashMap::<MyHasher>::new();
let map3 = HashMap::new(with_hasher: MySeededHasher::new(0));
```

I can give you this:

```rust
let vec1 = Vec::new();
let vec2 = Vec::new(in_alloc: MyAlloctor::default());
let vec3 = Vec::new(in_alloc: &my_local_allocator);

let map1 = HashMap::new();
let map2 = HashMap::new(in_alloc: MyHasher::default());
let map3 = HashMap::new(with_hasher: MySeededHasher::new(0));
```

Basically, I think we still have to hardcode the Type default still, but we can get the combinatorics of everything way more under control, making it way more tolerable to add new defaulted type parameters. In particular, you may have noticed that HashMap doesn't yet have an Allocator parameter. I don't know this for sure, but I bet it's because the combinatorics will *actually* get out of control at that point. Let's see how this approach scales.

We would end up with a HashMap implementation like this:

```rust
struct HashMap<K, V, S=RandomState, A=Global> { ... }

fn default_hasher() -> RandomState { ... }
fn default_alloc() -> Global { ... }

impl <K, V, S: BuildHasher, A: Allocator> HashMap<K, V, S> {
    fn new(
        with_hasher hasher: S = default_hasher, 
        in_alloc alloc: A = default_alloc,
    ) -> Self 
    { ... }

    fn with_capacity(
        capacity: usize,
        with_hasher hasher: S = default_hasher, 
        in_alloc alloc: A = default_alloc,
    ) -> Self
    { ... }
}
```

And be able to use it like this:

```rust
// All of these now work
let map1 = HashMap::new();
let map2 = HashMap::new(with_hasher: MyHasher::new());
let map3 = HashMap::new(with_alloc: MyAlloc::new());
let map4 = HashMap::new(
    with_hasher: MyHasher::new(),
    with_alloc: MyAlloc::new(),
);

// And we don't need 4 new copies of alternative constructors:
let map5 = HashMap::with_capacity(10, in_alloc: &local_allocator);
```

Note especially "map3" where we are "skipping over" the with_hasher argument. Without named arguments, this is annoying/impossible to express. If it *was* possible you would have to write something like `HashMap::new(_, MyAlloc::new())`, which, is going to scale incredibly badly.




# Solving Hashers and Allocators: Hacky Version

Ok here's where things start getting hacky just to see what happens. What if we wrap all our defaults args Options?

```rust
fn default_hasher() -> Option<RandomState> { ... }

impl <K, V, S: BuildHasher> HashMap<K, V, S> {
    fn new(with_hasher hasher: Option<S> = default_hasher) 
        where S: Default,
    {
        let hasher = hasher.unwrap_or_default();
        ...
    }
}
```

That seems pretty pointless! But, now you can write this:

```rust
// uses S=RandomState
let map1 = HashMap::new();
let map2 = HashMap::<MyHasher>::new(with_hasher: None);
let map3 = HashMap::new(with_hasher: Some(MySeededHasher::new(0)));
```

Basically, by adding an Option (or an enum equivalent to Option), the user can explicitly request a "partial application" of the default. In this way we can squeeze all 3 usecases into one function... kind of. The explicit map3 case takes two hits:

* It is now *extra* explicit
* It must now conform to Default *even though we're explicitly providing it*

I don't know how to do better than this without true changes to the type system. But, hey, it's there as an option?  Ok actually if you want to get Really Fucking Hacky we could make a MaybeDefault trait that basically panics for types that can't actually be defaulted or something?

```rust
trait MaybeDefault {
    fn maybe_default() -> Self;
}

impl<T: Default> MaybeDefault for T {
    fn maybe_default() -> Self { Self::default() }
}

impl MaybeDefault for MyUndefaultableType {
    fn maybe_default -> Self { unreachable!("get fucked") }
}
```

This is certainly Code That Can Be Written. Should it be written? Absolutely Not.



# Implementing Default With Optional Args

At the top of this post, I mentioned how it sucks that Vec::default is fucked up by introducing new defaulted type parameters. Can we fix this?

Not with my design! I initially thought you could maybe allow Extra Optional Arguments in trait impls because the compiler has to be able to generate the Proper One as a reabstraction thunk anyway... and I think that part is *plausible*, but it doesn't actually solve the problem.

You would end up with:

```rust
fn default_alloc() -> Global { Global }

impl<T, A: Allocator> Default for Vec<T, A> {
    fn default(in_alloc alloc: A = default_alloc) -> Self {
        Self::new(in_alloc: alloc)
    }
}
```

Which... doesn't actually do anything! Anyone who calls default() is just hardcoding the default, which is Global! You can't make default_alloc generic over `A: Allocator`, because then you get HashMap::default's need to be explicit and break everyone calling Vec::default(). Argh!

I've got absolutely nothing on this one, it seems like it really truly needs Defaults Affect Inference. 😭




# Conclusion

Whelp, I failed!

A lot of this stuff is really nice, and I would personally love to have it the language if only to tame the *combinatorics* of defaults a little bit. And for people who aren't *explicitly* trying to solve the Allocator problem, this gives them a lot of great tools for building nice APIs.

But I just don't see how this can be done without proper type/inference changes. Maybe all of this stuff makes the hard cases trivial? That would be nice. 

I didn't talk to any compiler/lang people while writing this. I also didn't talk to anyone on the allocators working group or really look over their notes. This is just a brain worm that has bugged me for almost a decade, and I was like, The Collections Person on the libs team, so I understand the problemspace pretty well! 

This was mostly just an excuse to write out the problem in full and mess aorund with some cool ideas that were just FUN TO THINK ABOUT regardless of the practical details! If either of those groups has cracked the nut on Defaults Affect Inference without me noticing, then, holy shit! Amazing fucking work!!

Also I mean... the hacky solution isn't that bad? I wouldn't have even pushed forward with allocators at all given the current situation, so clearly y'all are kinda ok with suffering!?

Is it that bad..?