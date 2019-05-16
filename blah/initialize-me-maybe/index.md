% Here's My Type, So Initialize Me Maybe

<span class="author">Alexis Beingessner</span>

<span class="date">May 16th, 2019 -- Rust Nightly 1.36.0</span>

Rust's infamous mem::unitialized method has been deprecated. Its replacement, MaybeUninit, has been stabilized. If you are using the former, you should migrate to using the latter as soon as possible. This was done because it was determined that mem::uninitialized was fundamentally broken, and could not be made to work.

Most of this post is dedicated to discussing the nature of uninitialized memory and how it can be worked with in Rust. [Feel free to skip to the details on why mem::uninitialized is broken][section-what-went-wrong].





# What's Uninitialized Memory?

When you allocate memory in Rust it arrives to you in an *uninitialized* state. What exactly that means is a surprisingly subtle issue.

From a low-level implementation perspective, this generally just means that some region of memory has been declared to be "yours", but because that memory could have been previously used for something else, there's no guarantee what the bits in that memory are set to. Why are old values from somewhere else still there? Because it's faster and easier to not bother to clear out the memory. If you read from uninitialized memory, you are essentially reading random bits, and so your program may behave randomly. From this perspective, it's almost always a serious bug to read from uninitialized memory. Although you could certainly construct cases where that's not the case.

Note that this model is one of a single process that recycles memory it is has acquired from the operating system without returning it. For security reasons, memory freshly acquired from your operating system is guaranteed to be initialized to all 0's. [Although there are certainly security-minded folks who would love for processes to do this internally as well][memset].

The theoretical perspective is a bit more strict, [and also very poorly specified][undef]. The long and the short of it is that compilers consider uninitialized memory to be a much more exotic thing. Something of a magical substance that turns into whatever value makes the compiler's life easiest. Would it be nice if x was 1 here, so you could apply the simplification `y & x => y`. Oh it's uninitialized memory? Ok let's assume it's 1.

Unfortunately, what compilers most love in the world is to prove that something is Undefined Behaviour. Undefined Behaviour means they can apply aggressive optimizations and make everything go fast! ...Usually by deleting all your code.

So, as a conservative model it's reasonable to just declare that **it is Undefined Behaviour to read uninitialized memory**. Full stop.





# Safely Working With Uninitialized Memory In Rust

Given a model where we must never read Uninitialized Memory, what tools does Rust give us?

Let's look at the simple case of a local variable. Normally, when you declare a local variable, you also initialize it:

```rust
let x = 0;
let (a, b) = (1, 2);
```

In this case, there's no concern for Uninitialized Memory. As soon as we're given the *ability* to refer to our new piece of memory, it has already been initialized. But Rust lets us write this code:

```rust
# fn some_condition() -> bool { false }
#
let x;
if some_condition() {
    x = 5;
} else {
    x = 7;
}
println!("{}", x);
```

Unlike [Java][java-default-init] and [C++][cpp-default-init], Rust does not have the concept of default-initialization. In this code x is initialized *exactly once*. However we clearly are allowed to refer to x before it's initialized. What if we change our program to try to read x when it could be uninitialized?

```rust
# fn some_condition() -> bool { false }
#
let x;
if some_condition() {
    x = 5;
}
println!("{}", x);
```


```text
error[E0381]: borrow of possibly uninitialized variable: `x`
 --> src/main.rs:8:20
  |
8 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

Aha, Rust statically prevents us from reading x when it could be initialized. Interestingly, this *does not* mean that we must always initialize `x`. This program compiles:

```rust
# fn some_condition() -> bool { false }
#
let x;
if some_condition() {
    x = 5;
    println!("{}", x);
}
```

In addition, rust can dynamically track the initialization of values with destructors (Drop impls),
with a system called *[drop flags][drop-flags]*:

```rust
#fn some_condition() -> bool { false }
#fn some_other_condition() -> bool { true }
#
let mut x;              // x may be dynamically init
                        // drop_flag(x) = uninit

if some_condition() {
    x = Box::new(12);   // statically uninit, init it
                        // drop_flag(x) = init

    println!("{}", x);  // statically init here, ok to read!
}

x = Box::new(13);       // dynamically init, check drop_flag(x):
                        // - if init: drop it, then init it
                        // - if uninit: init it
                        // drop_flag(x) = init

println!("{}", x);      // statically init, ok to read!

if some_other_condition() {
    std::mem::drop(x);  // statically init, ok to read
}
                        // dynamically init, check drop_flag(x):
                        // - if init, drop it
```

That lets the compiler know when destructors should be run, but doesn't allow us to actually work with dynamically initialzed values. We're still only allowed to insert an explicit read if the value is *statically* known to be initialized. For truly dynamic initialization, rust has the Option type (or any enum, really):

```rust
#fn some_condition() -> bool { true }
#
let mut x: Option<_> = None;    // Only init a tag in the enum
                                // indicating the value is uninit

if some_condition() {
    x = Some(36);               // Set the tag and init the value
}

if let Some(val) = x {          // Acquire the value in x if it's init
    println!("{}", val);
}

let y = x.unwrap();             // Assert the value is definitely init
                                // crash the program if not
println!("{}", y);
```

As a minor aside, all of this initialization logic is the main motivation for Rust's `loop { }` construct. The compiler can understand that it always runs, and if you ever exit the loop you must have hit a `break` or `return`. In this way it can more easily reason about paths of execution and initialization.





# Unsafely Working With Uninitialized Memory in Rust

All of the tools we have looked at before now have been completely safe, but as you may know, Rust has an [unsafe side][unsafe].

To my knowledge, Rust has 3 `unsafe` ways to acquire uninitialized memory that it can't prevent you from reading:

* [raw heap allocation][alloc]
* [untagged unions][unions]
* [mem::uninitialized][] (BUSTED AND DEPRECATED)





## Raw Heap Allocation

Calling `std::alloc::alloc` will give you a pointer to brand-new allocation, which means it's a pointer to uninitialized memory. To correctly work with this memory, you must carefully initialize it with [raw pointer methods][ptr-methods] like `write` and `copy_from`. These methods assume the target is uninitialized, and let you initialize it. It is up to you to maintain enough state to know when the memory is uninitialized. Ideally, you will also `read` or `drop_in_place` any initialized memory whose type has a Drop impl when you're done with it, although forgetting to is *technically* allowed.

Nothing too complex here.





## Untagged Unions

Untagged unions, on the other hand, are a little more subtle. For the most part, Rust treats unions the same as any other type. You can't just not initialize them. This still won't compile:

```rust
union MyUnion {
    case1: u32,
    case2: u64,
}

unsafe {
    let x: MyUnion;

    println!("{}", x.case2);
}
```

```text
error[E0381]: borrow of possibly uninitialized variable: `x`
 --> src/main.rs:9:20
  |
9 |     println!("{}", x.case1);
  |                    ^^^^^^^ use of possibly uninitialized `x.case1`
```

But the cases of our union have assymteric sizes. What happens if we initialize the small case, but read from the large one?

```rust
union MyUnion {
    case1: u32,
    case2: u64,
}

unsafe {
    let x = MyUnion { case1: 0 };

    println!("{}", x.case2);
}
```

```text
> 140720308486144
```

Whoops! Rust won't prevent us from doing this, and so we have a way to read uninitialized memory without needing to perform a heap allocation. Interestingly, we can take this to its logical limit and build UntaggedOption, which lets us dynamically initialize any value:

```rust
# #![feature(untagged_unions)]
# #[allow(unions_with_drop_fields)]
# fn some_condition() -> bool { true }
#
union UntaggedOption<T> {
    none: (),
    some: T,
}

unsafe {
    let mut x = UntaggedOption { none: () };

    if some_condition() {
        x.some = 7;
    }

    // Boy we better have taken the some_condition branch!
    println!("{}", x.some);
}
```

Unlike C++, Rust does not have the notion of an ["active" union member][cpp-active-union]. Nor does Rust have C++'s [strict type-based aliasing][cpp-strict-aliasing]. As such, Rust freely allows you to use unions for type punning. Just be careful not to read uninitialized memory! (including padding bytes)



## mem::uninitialized

Finally, we come to the focus of this post.

The intended semantic of `mem::uninitialized` is that it pretends to create a value to initialize some memory, but it doesn't actually do anything. In doing this, the static initialization checker becomes convinced the memory is initialized, but no work has been done. The motivation for this function is cases where you want to dynamically initialize a value in a way that the compiler just can't understand, with no overhead at all.

For the compiler people out there, `mem::uninitialized` simply lowers to [llvm's undef][undef].

Of course, you need to be careful how you use this, especially if the type you're pretending to initialize has a destructor, but you could imagine being able to do it right with `ptr::write` and `ptr::read`. For Copy types, it's seemingly not that hard at all. Here's the kind of program that motivated this feature:

```rust
# fn some_val() -> u32 { 7 }
unsafe {
    // Trick the compiler into thinking we initialized this!
    let mut results: [u32; 16] = std::mem::uninitialized();

    for i in 0..16 {
        // complex logic...
        results[i] = some_val();
    }

    // All values carefully proven by programmer to be init
    println!("{:?}", &results[..]);
}
```





# What Went Wrong?

Ok so you have determined that you're in a special case where you need to convince the compiler's safety checks that a value is initialized without actually initializing it. Say you write this:

```rust
# fn some_condition(i: usize) -> bool { i % 2 == 0 }
unsafe {
    // Trick the compiler into thinking we initialized this!
    let mut results: [bool; 16] = std::mem::uninitialized();

    for i in 0..16 {
        results[i] = some_condition(i);
    }

    // All values carefully proven by programmer to be init
    println!("{:?}", &results[..]);
}
```

For me, on the day of writing this, this program compiles and executes fine. Unfortunately, this program has Undefined Behaviour. Why? Because bool is a primitive that has *invalid values*. To quote the Rustonomicon, it is Undefined Behaviour to produce an invalid primitive value such as:

* dangling/null references
* null `fn` pointers
* a `bool` that isn't 0 or 1
* an undefined `enum` discriminant
* a `char` outside the ranges [0x0, 0xD7FF] and [0xE000, 0x10FFFF]
* A non-utf8 `str`

Remember when I said that compilers can magically make uninitialized memory any value they want? And how they want everything to be Undefined Behaviour? Well because we tell the compiler that a bool is either 0 or 1, if the compiler can prove that a value of type bool is uninitialized memory it has succesfully proven the program has Undefined Behaviour. No, it doesn't matter that we didn't read the uninitialized memory.

So although mem::uninitialized *can* possibly be used correctly, for some types it's *impossible* to use correctly. As such, we're tossing it in the trash. It's a bad design. You should use its replacement, MaybeUninit.




# What Is MaybeUninit?

In the section on [untagged unions][section-untagged-unions], I noted that in the extreme case you could make an UntaggedOption type:

```rust
union UntaggedOption<T> {
    none: (),
    some: T,
}
```

Well it turns out that's all that MaybeUninit is. Well actually, it's defined as:

```rust
pub union MaybeUninit<T> {
    uninit: (),
    init: ManuallyDrop<T>,
}
```

But that's it. The compiler doesn't even know about it as a special type. It's just a union with a dummy "uninit" case. With this, we can make our program correct:

```rust
# #![feature(maybe_uninit)]
# #![feature(maybe_uninit_ref)]
# fn some_condition(i: usize) -> bool { i % 2 == 0 }
#
use std::mem::MaybeUninit;

unsafe {
    // Tell the compiler that we're initializing a union to an empty case
    let mut results = MaybeUninit::<[bool; 16]>::uninit();

    // VERY CAREFULLY: initialize all the memory
    // DO NOT: create an &mut [bool; 16]
    // DO NOT: create an &mut bool
    let arr_ptr = results.as_mut_ptr() as *mut bool;
    for i in 0..16 {
        arr_ptr.add(i).write(some_condition(i));
    }

    // All values carefully proven by programmer to be init
    println!("{:?}", &results.get_ref()[..]);
}
```

Why is this different from when we used mem::uninitialized? Because the compiler can *clearly* see that our memory has type "either an array of bools, or nothing". So it knows not to assert that the memory must have any particular value.

Note that we must still be careful. While this isn't finalized, the preferred semantics of references allows the compiler to assume that they're non-dangling and point to valid memory. Under those semantics, if we create an `&mut [bool; u16]` or `&mut bool`, it could be Undefined Behaviour.

To avoid this issue, we only manipulate the memory using raw pointers, in the same way we would initialize a heap allocation. I wasn't 100% sure if I could claim that `arr[i] = x` doesn't create a reference, so I just used pointer arithmetic to be safe.

Have fun writing your terribly unsafe, but definitely, absolutely, rigorously proven correct programs!










[undef]: http://www.cs.utah.edu/~regehr/papers/undef-pldi17.pdf
[memset]: http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1381.pdf
[cpp-default-init]: https://en.cppreference.com/w/cpp/language/default_initialization
[java-default-init]: https://blog.ajduke.in/2012/03/25/variable-initialization-and-default-values/
[drop-flags]: https://github.com/rust-lang/rfcs/blob/master/text/0320-nonzeroing-dynamic-drop.md
[unsafe]: https://doc.rust-lang.org/nightly/nomicon/meet-safe-and-unsafe.html
[alloc]: https://doc.rust-lang.org/alloc/alloc/fn.alloc.html
[unions]: https://doc.rust-lang.org/reference/items/unions.html
[mem::uninitialized]: https://doc.rust-lang.org/std/mem/fn.uninitialized.html
[ptr-methods]: https://doc.rust-lang.org/std/primitive.pointer.html
[undef]: https://llvm.org/docs/LangRef.html#undefined-values
[cpp-active-union]: https://en.cppreference.com/w/cpp/language/union
[cpp-strict-aliasing]: https://blog.regehr.org/archives/1307
[section-what-went-wrong]: #what-went-wrong
[section-untagged-unions]: #untagged-unions
