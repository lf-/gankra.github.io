% The Kinds of Implementation-Defined?

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">November 13, 2018</p>
</header>

This post is intended to try to shake loose a distinction I found myself trying to make.

When designing a specification you will inevitably run into a corner case. The compiler world has generally adopted a 4-tier system for these non-obvious situations:

* **Defined Behaviour:** here's exactly what implementations should do (at least semantically)
* **Undefined Behaviour**: implementation may freely assume this *cannot* happen
* **Unspecified Behaviour**: implementations may do whatever ("soft" undefined behaviour)
* **Implementation-Defined Behaviour**: implementations must pick a solution and document it.

I could write a bunch about these different approaches and how they have internal sub-strategies, but today I want to focus in on the last one: implementation-defined behaviour.

In my mind there are two distinct reasons to adopt implementation-defined behaviour, and they're different enough that I want them to just be distinct categories:

1. This is modeling something inherently platform-specific, and so we need to be able to make this do different things on different platforms. But on all (or most) platforms, the decision is fairly straight-forward, and it would be strange for two implementations that target the same platform to make a different decision here. Let's call this **platform-defined**.

2. There's some kind of fundamental tradeoff here that reasonable implementations could disagree on. An implementation may even want to make this configurable since it's a policy decision best made by the end-user. Let's call this **configurable**.

Let's look at some examples.

The size of a pointer is a good example of something which is largely platform-defined. Once you know you're targeting x64 windows, you know the size of a pointer should be 64-bit. But we consider this "implementation-defined" because we lack that level of nuance.

Many other implementation-defined choices in C are pretty clearly concessions to things which might reasonably (at the time) vary from platform-to-platform: the value of NULL, the size of int (intended to be the platform's "native" integer size), what a float even is, and so on. In many of these cases, the OS or hardware tend to make these decision pretty straight-forward. (Except for int on 64-bit systems, which has ended up being more defined by backwards compatibility concerns, but that is in some sense still an aspect of the platform.)

ABIs, which generally directly reference hardware-specific elements like registers, are also necessarily implementation-defined. But at the same time, if you want to interoperate with your OS's libraries, you better stick to their definition!

At the moment the best examples of "configurable" I can think of are what has become of some of the more controversial instances of Undefined Behaviour in C. Major compilers have moved towards a model where flags can be provided to make some of these normally undefined behaviours *only* implementation-defined.

Althought signed integer overflow is Undefined Behaviour, the major C compilers let users specify implementation-defined strategies instead. Specifically, developers may choose "trap on overflow" (ftrapv) and "wrap on overflow" (fwrapv). Integer overflow is a problem no one can agree on with very real tradeoffs, while also having little impact on modern platform-compatibility. Making it implementation-defined in this way is a reasonable strategy.

The Rust team actually adopted this strategy as part of the language specification: overflow is *defined* to either wrap or panic, and that should be defined by the compiler (also the panic may be deferred slightly by an implementation to allow overflow checks to be coalesced for better codegen).

In rustc, this is accomplished by:

* panic in debug (so hopefully you catch it in testing)
* wrap in release (in the hopes that it's benign, and also to get better codegen)
* A `-C overflow-checks=on|off` flag to explicitly request panics (on) or wrapping (off)

Anyway, that's just my thoughts on implementation-defined behaviour. What do you think?

