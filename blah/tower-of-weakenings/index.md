% The Tower of Weakenings: Memory Models For Everyone

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">April 4th, 2022</p>
</header>


In my [previous article](https://gankra.github.io/blah/fix-rust-pointers/) I looked at the ways that Rust could be *better* at pointers. Half of the problems were *fundamental* issues that affect basically every programming language that lets you mess with pointers directly, and the other half were rust-specific issues. My article also proposed two solutions: for the *fundamental* issues I proposed "strict provenance", and for the *specific* issues I proposed the \~ operator.

I knew talk was cheap and that no one would *actually* implement them, but I still wanted to get the ideas out there, especially for syntax changes! But wait, the solution to the fundamental issues didn't have any syntax. Actually it was essentially adding 2 completely trivial functions to the standard library.

So [I just did it](https://doc.rust-lang.org/nightly/std/ptr/index.html#strict-provenance), because I could. And [I ported most of Rust's codebase to it](https://github.com/rust-lang/rust/pull/95241) to prove that it actually works. And I made [a stable polyfill](https://docs.rs/sptr/latest/sptr/index.html) so that everyone could use it right away. And then Ralf Jung went off and added a [mode to miri to check it](https://github.com/rust-lang/miri/pull/2045)! **(HEY USE ALL OF THIS!)**

The actual *feature* was basically trivial. I made more work for myself by insisting on porting rust's codebase to the APIs, but that was just to Prove My Point. The *real* work was just endless discussions to convince stakeholders, and writing [an absolute metric fuckton of documentation](https://doc.rust-lang.org/nightly/std/ptr/index.html#strict-provenance) on "the model".

Shockingly, the core ideas in my article essentially held true all the way through this process, and the core APIs are [there](https://doc.rust-lang.org/nightly/std/primitive.pointer.html#method.addr) exactly as [proposed](https://doc.rust-lang.org/nightly/std/primitive.pointer.html#method.with_addr). What *did* evolve was the depth of the concepts, and how I communicated them, and *that's* what this article is really about.

*Really* late into the process I stumbled across an extremely powerful way of framing Strict Provenance: **The Tower Of Weakenings**. I hastily added it [to the FAQ](https://github.com/rust-lang/rust/issues/95228#issuecomment-1075881238) right before announcing the feature, but now that I have some more room to breathe, I want to really give it a chance to shine on its own.

Just to have it upfront, I will start with what The Tower Of Weakenings *is*, and then go back to the motivation and ideas behind it. The Tower Of Weakenings is simply the idea that having One True Memory Model is a frustrating and futile endeavour that leaves no one satisfied. Instead, we should have a *tower* of Memory Models, with the ones at the top being "what users should think about and try to write their code against". As you descend the tower, the memory models become increasingly complex or vague but *critically* always more permissive than the ones above it. At the bottom of the tower is "whatever the compiler actually does" (and arguably "whatever the hardware actually does" if you care about that).

Here is a sketch of what the tower looks like under my proposal:

1. The "Clean" Memory Model: a painfully strict and simple model that you can teach and check.
2. The "Real" Memory Model: similar to strict provenance, but messier to allow for Useful Crimes.
3. The Compiler's Semantics: whatever random primitives compilers expose, and optimizations they do.

In some sense the bottom of the tower is the one that "matters" because that's the thing that (mis)compiles your code, but it's also a shifting target, and [trying to expose its semantics leads to sadness](https://gankra.github.io/blah/initialize-me-maybe/). Unfortunately, this is also basically true of "real" memory models. [We're still trying to figure out what on earth C's memory model is](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2676.pdf), and every language infamously defers to "the C11 memory model" on hard questions. So as an educator, I eventually have to bottom out on sending you a shrug-emoji if you keep asking for what the *real* rules are. No one *really* knows!

Even relatively rigorous things like CompCert are just [doing their best](https://arxiv.org/pdf/2201.10280.pdf). We're all just doing our best. I started becoming acutely aware of this around 2015 when I [tried to document Rust's semantics](https://doc.rust-lang.org/nightly/nomicon/) and had to leave [the section on aliasing](https://doc.rust-lang.org/nightly/nomicon/aliasing.html) as a big 🤷‍♀️. Since then I have been slowly working my way through the stages of grief, and The Tower Of Weakenings is *Acceptance*. 

Memory Models are messy, and that's ok, because I will give you a simpler model. It will be teachable. It will be simple to reason about. It will be machine-checkable by sanitizers like [miri](https://github.com/rust-lang/miri). It will be overly strict, and that's ok, because if you want you can always take a step down the tower of abstractions and follow its rules instead. You will just have to deal with the reality of everything getting fuzzier, and the fact that you don't get as nice tools for catching your mistakes. 

There is always a Tower of Weakenings, and we're all messing with it, so why not just be honest about it and use it to our advantage? Like, once you get to the level of things like The Linux Kernel you start finding things that are blatantly breaking the rules of friggin' *Intel CPUs*, because hey it turns out there's a sub-basement of the Tower of Weakenings that's just "look this works on every CPU I can find, and why would this possibly break, and also I'm Linux so if your CPU doesn't run me *you're* the asshole, so you *can't* break it now".



# Memory Models Are Being Asked To Do Too Much

Ok yes get your jokes out about solving all problems with another layer of abstraction, but this is *painfully* needed. And really, what I'm doing here isn't *adding* a layer but cleaving a bloated and messy layer into two much more manageable parts. No offense to all the people doing great work on memory models, but as far as I'm concerned they've been given an impossible task. There are simply too many competing concerns and stakeholders to produce a *truly* satisfying design for all of them. Here are some of the many stakeholders:

* Millions of lines of ancient code that is doing whatever because we can't explain the rules
* Random programmers who barely read the docs and are just doing The Idiomatic Thing
* Hardcore performance junkies who insist the fucked up thing they made should be legal
* Hardcore safety junkies who insist they should be able to validate the correctness of their code
* Compiler developers who want to know what optimizations they can or can't do 
* Libarary developers who want to know what interfaces/idioms to design around
* Tooling developers who want to build sanitizers that reliably detect UB when it happens
* Teachers who want to help everyone else understand this PhD thesis you just dumped on their desk.
* Memory model people who just want this all to be coherent and formalized and validated

And all of these people are working *together* so they all agree that whatever the other person claims they need to do their job should definitely be allowed, because they want to all mutually benefit. And definitely make sure not to break all that code or even make it run slower! kthxbai!

Look if you do Memory Model stuff and think this is tractable. Fucking rad, I hope you succeed, nut I am willing to call this "absolutely fucking impossible". You have been given a Shitty Gordian Knot and I am here to offer you the gift of my sword slicing it in half so we can all get on with our lives and make computers less terrible.

The new layer I am adding to The Tower Of Weakenings rejects the possibility of serving all those concerns, and instead focuses on giving everyone a *Really Great Lie* that we can all agree to pretend is true and make everyone's job easier. For everyone who doesn't like the Lie, they can either admit they don't care and hope to not get miscompiled (as we have all been doing already) or they can walk down the steps of the tower and start [reading some academic papers](https://plv.mpi-sws.org/rustbelt/stacked-borrows/).

Critically, the fact this lie *is* part of The Tower Of Weakenings means that if you follow its rules, then whatever the "real" model is *doesn't matter anymore*. We will absolutely guarantee that your code works, because you followed the really strict and simple rules that *everyone* can agree *obviously* have to work. Just as you're not supposed to worry about the compiler backend you're using, you shouldn't have to worry about the fiddly details of what the current working memory model is.

Let's look at how this addresses the concerns of the many stakeholders:

* Ancient code: the real rules haven't changed, so it's just as fine as it was yesterday! Although people might do audits or run tools over it and notice that it doesn't follow the Clean Rules and either fix it, abandon it, or mark it as "yeah we know". All of these are better than the status quo, imo.

* Random programmers: probably unaffected, but The Idiomatic Thing is now more bulletproof, so that's good.

* Hardcore performance junkies: mildly sad, but at least have explicit promises that more evil stuff *can* work and that the "real" model/compiler generally will do its best to allow it. Will at least get new conveniences to make "staying in the clean model" easier and more ergonomic. (I have [done this for you before](https://github.com/rust-lang/rfcs/pull/1966), I [did it for you here](https://doc.rust-lang.org/nightly/std/primitive.pointer.html#method.map_addr), and I am [already pushing for more](https://github.com/rust-lang/rust/pull/95643). I will not rest until everyone says "I wish C made it as easy to use raw pointers as Rust does", and I am not joking.)

* Hardcore safety junkies: hell yes they love [stricter subsets](https://www.misra.org.uk/) that they can mechanically enforce and validate. They get better tools for doing that validation too (see Tooling below)! Safety Engineer Christmas!

* Compiler developers: business as usual since they only care about the "real" model, but hey evil code is a lot more rare now, so less people are running into nasty edge cases.

* Library developers: some mild sadness from everyone now insisting that some code is "bad" and should be avoided/destroyed. In exchange, they get idioms they can build around that everyone can agree is legit. Also get better tools for increasing confidence in hairy unsafe code (see Tooling below). (This is me, I am library developers. I have the dubious honor of being responsible for [one of Rust's only CVEs](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-1000657) because trying to keep everything in your head all the time as a horrible meat golem is exhausting and impossible! Give me better tools!!)

* Tooling developers: tools can be [more precise](https://github.com/rust-lang/miri/pull/2045) by default, because more code is "trying" to conform to the strict model. [Miri](https://github.com/rust-lang/miri) gets to be Perfect [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) and more reliably catches important things! Yesssss

* Teachers: hell yes there's an actual model that is simple enough to actually teach people, and they can even tell people that the tools for catching mistakes are *good* and will actually help you catch mistakes! (This is also me, [I](https://doc.rust-lang.org/nightly/nomicon/). [Am](https://rust-unofficial.github.io/too-many-lists/). [All](https://doc.rust-lang.org/std/ptr/index.html). [Of](https://doc.rust-lang.org/std/collections/index.html). [Your](https://gankra.github.io/blah/rust-layouts-and-abis/). [Docs](https://gankra.github.io/blah/initialize-me-maybe/). [And](https://gankra.github.io/blah/hashbrown-tldr/). [You](https://gankra.github.io/blah/only-in-rust/). [Will](https://gankra.github.io/blah/linear-rust/). [Learn](https://gankra.github.io/blah/everyone-poops/).)

* Memory Model People: The Gordian Knot Has Been Destroyed. You can make your shit as complicated and messy as you need it to be, because most people have agreed not to ask too many questions and just use the simple and strict thing that will absolutely be a subset of whatever you do. Use this power responsibly, it would still be genuinely good if all the terrible ancient code still worked.

As far as I am concerned, The Tower of Weakenings is horrifyingly effective. Like really it's actually fucked up that I can hand out wins like this by just saying "but what if we just all agree to pretend things are simpler most of the time".




# Ok But What The Hell Is Strict Provenance

You're really not gonna go read any of [all](https://doc.rust-lang.org/nightly/std/ptr/index.html#strict-provenance) the [docs](https://github.com/rust-lang/rust/issues/95228) I [wrote](https://github.com/rust-lang/rust/issues/95228#issuecomment-1075881238) about [it](https://gankra.github.io/blah/fix-rust-pointers/#replacing-pointer-integer-casts) are [you](https://twitter.com/Gankra_/status/1509335249871900678)? Ok it's fine. It's fine. I write the docs. I will write more here.

Here is the 6 step summary of what the model is.

1. Use [this library](https://docs.rs/sptr/latest/sptr/index.html).

2. Stop using `ptr as usize` and `usize as ptr` -- use `ptr.addr()` and `ptr.with_addr(addr)`.

3. Test your code with "cargo [miri](https://github.com/rust-lang/miri) test -Zstrict-provenance"

4. Report any nasty problems you run into [on the tracking issue](https://github.com/rust-lang/rust/issues/95228), or [Zulip](https://rust-lang.zulipchat.com/#narrow/stream/136281-t-lang.2Fwg-unsafe-code-guidelines), or contact your local congressman.

5. Insist that your evil code is special and good, use `ptr.expose_addr()` and `ptr::from_exposed_addr(addr)` instead to at least document this assertion and help us identify Fun Things.

6. Actually just ignore all of this because you're busy and tired and have so much dogshit unsafe code you "own" (same, I hate it [here](https://crates.io/crates/linked-hash-map/0.5.4)!). Incrementally update your code to the new more clear system whenever you're poking in there, or just let someone else file a PR because you're blowing up miri for their project that depends on you. Rest safe in the knowledge that Strict Provenance Is All Lies and that the language is exactly the same as it was yesterday, and will remain the same tomorrow too.

Yes that sure didn't explain anything at all. That was actually "what do I do", which is actually the important thing. Did you follow all 6 steps? Ok great, here's an actual explanation:

-----

If you cast an integer to a pointer, that's ok, it's fun to play pretend, but that pointer is *invalid*. That means reading/writing through that pointer or using [offset](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset) on it is Undefined Behaviour. ([wrapping_offset](https://doc.rust-lang.org/std/primitive.pointer.html#method.wrapping_offset) is fine, and the usual [ZSTs](https://doc.rust-lang.org/nomicon/exotic-sizes.html#zero-sized-types-zsts) Get To Lie rules apply.)

If you're doing FFI, just make sure that the API is "right" from Rust's perspective. This may involve tweaking [extern decls that lie](https://github.com/rust-lang/rust/issues/95496) about pointers. 

If you're doing MMIO stuff, you're basically outside the memory model already but [we're working on figuring it out](https://twitter.com/Gankra_/status/1509343864775131141).

The golden rule is that the union of `int | ptr` is `ptr`, not `int`. ABIs generally all agree that pointers and pointer-sized-integers should have identical ABIs, because this punning is super common... unless your on an m68k, which thinks [you're a sinner either way](https://trofi.github.io/posts/191-ghc-on-m68k.html) (at least for return values). God nothing can be simple, can it?

Yes even [do this for AtomicUsize and AtomicPtr](https://github.com/rust-lang/rust/issues/95492).

That's it. Just say things that are pointers are pointers.

The next section will discuss *why* we care about that.



# Strict Provenance: No More Getting Lucky

Without proper memory models formalizing things like "what is a pointer", compiler backends can very innocently introduce several optimizations that are all fine in isolation but [can rube-goldberg machine into miscompilations when combined](https://www.ralfj.de/blog/2020/12/14/provenance.html).

These situations are especially nasty because you can't obviously just point at one optimization and say "oh it's that one, that one is bad". How do you decide which seemingly-innocent optimization is a problem? This is what a memory model is for. 

You define an "Abstract Machine" that your language is "emulating" the semantics of, and then optimizations which change the observable behaviour of "the abstract machine" are "bad". This is basically the same situation as a SNES emulator glitching out and flipping Mario upside-down. You generally want your emulator to faithfully reproduce the game! 

Conversely, having a memory model also defines the scope of what programmers are allowed to expect to work, and the compiler doesn't have to care about somehow trying to faithfully reproduce things that don't work on the Abstract Machine. Returning to SNES emulators, this is like refusing to support romhacks that do things that wouldn't work on the actual hardware, because you have no idea what on earth that even means. Harsh, but fair!

Things that "can't work" on the Abstract Machine are Undefined Behaviour. One thing we generally all agree should *definitely* be Undefined Behaviour is Use After Frees, because it makes no sense, is non-deterministic, and is a huge source of security vulnerabilities.

Ok so let's make that rule part of our Abstract Machine.

...how do you do that? *Can* you do that??

Let's think it through:

1. Allocate some memory
2. Get a pointer into that memory
3. Free the memory (dangling pointer)
4. Allocate some more memory
5. *GET LUCKY* and have your dangling pointer point into it
6. Read through the pointer (use after free)

How do you define a machine where this isn't "allowed"? One option is to define the machine to never reuse addresses, so you can never Get Lucky. This is a real thing some systems try to do, but most systems just aren't like that. Is that a problem? Are we breaking our promise to faithfully emulate the Abstract Machine if we reuse allocations? You can't *reliably* observe it, but whenever you Get Lucky you can. But maybe that's *fine* because it's Undefined Behaviour to dereference the pointer, so it's the *programmer's* fault if they observed it! Yeah!

Wait, does that mean it has to be Undefined Behaviour to check if a "dangling" pointer is equal to another one? Oof, saying pointer equality is UB is rough. Like sure we can all agree [inequalities between pointers from different allocations should be unspecified](https://en.cppreference.com/w/cpp/language/operator_comparison#Pointer_comparison_operators), that's Totally Natural And Fine, but *equality*? Say it ain't so!

...Ok so you can see how this gets really wonky really fast right?

As I understand it, the consensus solution to this is to just *admit* that stuff like banning Use After Frees implies pointers are kinda magic, and you just say they "know" the allocation they point into and aren't allowed to access any other memory. Use After Frees are then forbidden on the premise that a reallocation is a "different" allocation, and so any dangling pointers that happen to point into it can't access it. No Getting Lucky. Well, you can get lucky enough to notice two pointers are "randomly" equal, but you don't get to do anything scary with that information because your dangling pointer is still "invalid".

The idea that pointers just "know" what allocation they're part of is *provenance*. So, cool, pointers have provenance in the Abstract Machine, problem solved!

...huh? What's that? You want to turn a pointer into an integer? Ok well that's f- *AND TURN IT BACK INTO A POINTER?* Oh. My. Um. Did you want all integers to have provenance too? That has some very large implications for like... x==y not letting you substitute x for y anywhere in your optimizer, because they might have different provenance even if they compare equal. Yeah that sounds horrible to me too.

Ah ok, but the C folks seem to have cracked this one. If you hop on over to this 123 page paper, you'll see the [PNVI-ae-udi](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2676.pdf) model, which at a high level roughly says:

* The Abstract Machine has a global list of all "exposed" allocations
* Allocations come into the world unexposed (unaliased)
* Casting a pointer to an integer exposes the allocation for that pointer's provenance
* Casting an integer to a pointer does *a global hittest on all exposed allocations*
* If you hit an exposed allocation, congrats! You get provenance to it.
* If you hit *two* exposed allocations (because of one-past-the-end), be sad and tell the programmer to "just be consistent" and only access one of them.

This makes a lot of sense! Basically it's saying if you cast a pointer to an integer, the compiler has to throw up its hands in disgust and just accept than any integer gets to be cast into a pointer into that allocation. That's... pretty tame? Assuming you don't need to allow other operations to expose.

But also... it lets you write this:

1. Allocate some memory
2. Get a pointer into that memory
3. Cast it to an integer (A)
4. Free the memory
5. Allocate some more memory
6. Also expose that one somehow
7. Cast (A) back into a pointer
8. *GET LUCKY* and have your pointer now get blessed with provenance over the new allocation

This is just a Use After Free with extra steps! Trying to allow for int-to-ptr casts just reintroduces the concept of Getting Lucky, and that *sucks*. It might be a necessary evil, but it still makes me sad.

It also makes sanitizers really sad. The more you crank up "things that are allowed to expose addresses" (Hmm, doesn't memcpy operate on `char*`? Isn't that an integer..?), the more flexible a sanitizer needs to be about things that Should Definitely Be Undefined Behaviour, But I Have To Assume You're Very Smart And Perfect Because Integer Casts Say So. Which is to say, you can have a dynamic execution that everyone 100% agrees should be a Use After Free, and which you would therefore hope a sanitizer to catch... and it just has to shrug it's shoulders and assume it isn't.

On the flip side, if you say "screw that", your sanitizer will constantly set off false alarms because everyone is told that int-to-ptr casts are totally fine and cool. This also sucks!

This whole mess is why Strict Provenance is such a nice simplification to a memory model: it just puts it foot down and says No Absolutely Not You Do Not Get To Get Lucky.

But of course, Strict Provenance is "a lie", so you still can get lucky, so what's the point? The point is that if everyone agrees to just *pretend* that you're not allowed to do ptr-to-int casts, then having our sanitizers default to the strict semantics *is actually useful and effective*! Programmers that opt into this game also don't have to worry about "are we doing PNVI-ae-udi" or something else, because... it doesn't matter! Tower Of Weakenings Baby! Allowing any kind of int-to-ptr shenanigans is *definitely* weaker than forbidding them completely!

But ok we have all this code that does pointer crimes, this is useless! It would be, if we didn't have the glorious ~~hack~~ innovation of with_addr! This is the part of strict provenance that makes everything "work":

```
fn with_addr(self, addr: usize) -> Self
```

This is basically the exact same thing as `usize as ptr` *BUT* instead of being a unary operation, it's binary. You need to bring along a pointer with the provenance you *think* your new pointer should have, and then we use it to properly "rebuild" the pointer, unambiguously re-establishing the provenance chain of custody, hooray! Toss in some conveniences like `map_addr` and you've got yourself an actually tolerable system for still doing tagged pointer crimes while staying in the confines of Strict Provenance:

```rust
unsafe {
    // A flag we want to pack into our pointer
    static HAS_DATA: usize = 0x1;
    static FLAG_MASK: usize = !HAS_DATA;

    // Our value, which must have enough alignment to have spare least-significant-bits.
    let my_precious_data: u32 = 17;
    assert!(core::mem::align_of::<u32>() > 1);

    // Create a tagged pointer
    let ptr = &my_precious_data as *const u32;
    let tagged = ptr.map_addr(|a| a | HAS_DATA);

    // Check the flag:
    if tagged.addr() & HAS_DATA != 0 {
        // Untag and read the pointer
        let data = *tagged.map_addr(|a| a & FLAG_MASK);
        assert_eq!(data, 17);
    } else {
        unreachable!()
    }
}
```

Hooray!

Oh also strict provenance is basically actually implemented in hardware by [CHERI](https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/), so uh, doing this *also* just makes your code portable to CHERI. (Which also means the CHERI people are willing to help make this experiment work, thank you CHERI people you have been very helpful!!!)

Strict provenance does a lot of things for a lot of people. It's Kind Of A Big Deal, IMO.





# Strict Provenance News

Since the MVP shipped, [we've made a couple changes](https://github.com/Gankra/sptr/pull/3#issuecomment-1087089454). Those are my "release notes" for the change, but here's a summary:

Ralf decided it would be best to make it more explicit as to whether you're actually playing this game or not. In [his PR](https://github.com/rust-lang/rust/pull/95588) he added APIs that explicitly use the "expose" terminology and some vague gesturing to the same kind of thing the C proposal is doing -- which is basically all we would have been able to tell you before strict provenance too. The new APIs are:

* `fn expose_addr(self) -> usize`
* `ptr::from_exposed_addr(usize) -> *const T`
* `ptr::from_exposed_addr_mut(usize) -> *mut T`

And these are 100% semantically identical to `ptr as usize` and `usize as ptr`, while also actually telling us that you understand this is a thing and you are relying on. The other change he made (which I was dubious of at first) was to mark `addr()` as *actually* not exposing the address. Which is to say that `ptr::from_exposed_addr(my_ptr.addr())` doesn't "work". This *technically* makes it more dangerous to use `addr`, but it's still a safe function for the usual reasons of "it's only a problem if you actually try to deref/offset the resulting pointer".

I asked him to explicitly mark in the docs that this UB is not actually exploited right now and if this ends up being a mess we're willing to weaken it back to be identical to expose_addr. The motivation for marking it UB is twofold:

1. Having an *actual* semantic distinction allows miri to operate in a "mixed" mode where code that is playing along with strict provenance isn't exposing addresses and will generally be more strictly checked, but code which *isn't* playing along gets more leeway and we still let you Get Lucky sometimes.

2. It justifies the kind of optimizations llvm already does today, which are definitely unsound absent any proper provenance model. In particular something like PNVI-ae-udi effectively bans eliminating "useless" ptr->int->ptr roundtrips because it makes the compiler "forget" an address was exposed and make wrong inferences. Making addr explicitly not expose the allocation makes that transform sound!

