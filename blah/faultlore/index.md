% Faultlore: Learning Through Errors

<header>
    <p class="author">Aria Beingessner</p>
    <p class="date">February 27th, 2022</p>
</header>

This is an article about how I learn, how I teach, and why the fuck my website is called *Faultlore* now. My writing is always a rambling stream of consciousness, but this one is *really* just all over the place because this is like, trying to make sense of my very existence.

My mind lives in constant terror of ambiguity. I am often paralyzed with fear whenever I encounter something ambiguous, even if it's mundane. If I say "do you want lasagna tonight or something else?" and you reply "yeah sure", I may get anxious and demand your response be clarified before I do anything else.

This isn't a smartass contrarian thing. Human language is full of implicit context and meaning and that's cool and great! I'm usually pretty sure that you meant "yes let's do lasagna"! But there's just this broken part of my brain that is obsessively probing for ambiguity and inconsistency and it has gotten very good at screaming "YOU HAVE TO GUESS, WHAT IF YOU GUESS WRONG???"

I think this brain gremlin has genuinely shaped my entire adult life and career. It's *all* just a horrible obsession with ambiguity.

At first I fled from ambiguity and buried myself in academia and theory, where any complexity and ambiguity could be hand-waved by the "model" of computation you assumed -- gotta love that extremely unreal [Real RAM model](https://en.wikipedia.org/wiki/Real_RAM)!

But cowering from ambiguity grew tiresome. I could do more. *I could fight back.* **I could destroy ambiguity.**





# Constructive Contradictions

Ambiguity is a wily creature that hides in the shadows. If we truly wish to destroy it, then we can't just be happy when we don't see it -- *we must hunt for ambiguity and assure its nonexistence*. For me, this meant developing an absolute *obsession* with contradictions and counter-examples (and making me an even more insufferable conversational partner).

I can't really put this in coherent terms but thinking about things in direct and positive terms now just feels... *wobbly*. On the other hand, viewing things in terms of the counter-examples and contradictions feels *solid*.

To try to make this a bit more concrete, think of trying to describe how some system works as if you were trying to describe a shape you had drawn.

Telling me what *should* happen is like telling me that some particular points are inside the shape. This is useful, certainly, but I still can't picture the shape. Are those points representative of the general shape? Is there a weird sharp corner over there that I'll get impaled on? Who knows!

But if you tell me what *shouldn't* happen then suddenly it's like you're describing the outline of that shape to me. Every counter-example and error is another stroke of the boundary. It all becomes simple and clear.

I'm like... an anti-[Constructivist](https://en.wikipedia.org/wiki/Constructivism_(philosophy_of_mathematics)). If you just show me an example of something working, I won't trust you for a second. But if you tell me all the ways something *doesn't* work then I'll follow you to the ends of the earth.

This was very prominent in how I approached theorems and proofs as an academic. Rather than proving things *directly* I always found myself drawn to proving them *by contradiction*, which involves arguing that everything completely breaks if something *isn't* true, so it *must* be true (or the system we are using is completely broken, [which we all generally agree is rude to talk about](https://en.wikipedia.org/wiki/Consistency) (or you're an [Excluded Middle](https://en.wikipedia.org/wiki/Law_of_excluded_middle)/Constructivist pedant)).





# Aside: The Greatest Moment of My Academic Career

The greatest moment of my academic career was when my supervisor Dr. Michiel H. M. Smid drank a grad student under the table and hollered "FUCK YOUR FUCKING HASHTABLES". It will live in my heart forever.

The second greatest moment of my academic career was when I contradictioned so hard that I created a constructive proof. This would ultimately result in the greatest moment of my academic career.

I was writing my first paper, and trying to figure out whether a degenerate corner case could happen. My supervisor Dr. Smid called me into his office, asserting that he had a proof that it couldn't happen, which he started to walk me through.

It was a proof by contradiction. Since we were trying to prove that something couldn't happen, he started by assuming it could. This meant there must be some arrangement with such-and-such property, and so on and so on. But one step didn't make sense to me. I pointed out a possible counter-example for one of his steps... and I was right!

Since we were in contradiction land, the counter-example I had found was actually just... an example! An example that had degenerate property we thought couldn't happen! Suddenly we had gone from "I think this is impossible" to having a clear and simple constructive proof that it can!

Let me break that down: we were completely wrong about this problem, thought we had solved it with a proof-by-contradiction, refuted that proof with a counter-example, and got a constructive proof of the opposite result! 3 wrongs *did* make a right!






# Learning The Forbidden Knowledge

Because I spent so long soothing my broken brain on algorithms and data structures but was losing my interest in academia, I started trying to actually implement them. I initially tried to do this in C++, but I was (and honestly still am) just absolutely horrible at using C++. I love the *ideas* in C++, but the actual realities are just, so counter-intuitive to me.

But hey, I had heard of this new language "Rust" which is kinda like C++ but safer, and it even has syntax for unique_ptr and shared_ptr so using those must be way less painful than C++!

I wanted to know how to make collections in Rust, so I looked at how the standard library implemented them.

Badly, as it turns out.

I was an academic who knew nothing of the language, so under normal circumstances I would have no particular insights on the quality of the code... but this was pre-1.0 Rust after many years of messing around with different approaches to ownership. You know what really fucks up the way you represent collections of things? Completely changing the way your language talks and thinks about pointers and values.

Remember `@` and `~`? The standard library and compiler sure fucking did, and were covered in scars from their ongoing removal. (At the time I found this disappointing, that was the syntax I was excited to use!)

It doesn't take a veteran systems programmer to see obviously jacked up shit and fix it. TreeMap's `get` and `get_mut` had two completely different implementations. VecDeque was a wrapper around `Vec<Option<T>>`. Removing any value from BTreeMap consumed the whole tree by value and destroyed it. It was dire in there.

So I started asking some obvious questions and cleaning things up. And then I started asking some less obvious questions and doing some major overhauls. And then I started asking some really messy questions and wrote some fucking treatises.

Why did things get so hairy and complicated? Well it turns out that working on data structures in the standard library brings together basically every single feature and concept in the language? Even the parts that aren't like, officially part of the language, because standard libraries are generally slightly magical and get to do secret bullshit collabs with the compiler.

Look here's some fun secret bullshit the stdlib does off the top of my head:

* [There is a "DerefMove" operation that only Box supports.](https://doc.rust-lang.org/std/boxed/index.html) It's not really properly documented because it's kinda a weird api design boondoggle but they also can't remove it!
* [The compiler has special hacks to trust Vec's destructor to allow more jank stuff to compile](https://github.com/rust-lang/rust/blob/035a717ee8bf548868fb50b5c7ca562fc4a657a7/library/alloc/src/vec/mod.rs#L399)
* [Both of these types contain a special "owning pointer" type](https://github.com/rust-lang/rust/blob/035a717ee8bf548868fb50b5c7ca562fc4a657a7/library/core/src/ptr/unique.rs)
* [Why does drop_in_place just call itself? Magic!](https://github.com/rust-lang/rust/blob/master/library/core/src/ptr/mod.rs#L185-L194)
* [What the fuck is this stuff at the bottom of std? Load-bearing bullshit!](https://github.com/rust-lang/rust/blob/035a717ee8bf548868fb50b5c7ca562fc4a657a7/library/std/src/lib.rs#L623-L635)

But most importantly, data structures are the most fundamental and voracious consumers of *Unsafe Rust*. You know what happens when you fuck up Unsafe Rust? Undefined Behaviour. You know what happens when you get Undefined Behaviour? No, you don't. It's fucking *UNDEFINED*.

So it's really important that you understand what you're doing when you write Unsafe Rust.

You don't know what you're doing?

Look just pull up the documen-

Where the fuck is the documentation. No I see they wrote words but that's not actually documentation. You can't actually use what it says to write correct programs!

This is all so... horribly...

**AMBIGUOUS**





# The Lore

To resolve the ambiguity in my own mind, I needed to consume more knowledge. But there wasn't any kind of specification or reference. The knowledge I needed was scattered around the lands -- in the hushed whispers of the compiler devs; in the hastily scrawled comments in the issue tracker; in the strange angles and contortions of the code.

That which I sought after existed only as *folklore*.

These traditions and practices spoke of success, but echoed something else. The ancient rituals and ceremonies *worked*, but their very existence suggested that which didn't. I saw well-trodden paths that strangely curved around nothing, testaments to unspoken horrors that once took place there.

I sought the shadows cast by the folklore -- the forbidden acts that could bring doom, that had brought doom, and would bring doom again.

I sought the folklore of failure.

I sought the faultlore.

I voraciously consumed this knowledge, slaking my thirst at its altar and learning more of its secrets than any one person should. But as my pangs of hunger faded, I found not peace, but fever.

That which filled me now gnawed at my mind, for faultlore is a living thing, and that which lives desires a form by which it can be Known.

I obeyed the will of the abyss inside me and [carved its name into our reality](https://doc.rust-lang.org/nightly/nomicon/), desperate to release its grip on my thoughts. In doing so I found peace and clarity, and awoke with my knife plunged in the heart of the beast called ambiguity.

I also got a free laptop.

And the ability to win every argument on the internet by citing myself like a tenured professor who mentally checked out 10 years ago.







# Learn By Contradiction

A recurring theme you will find in The Rustonomicon is *case studies of places where we fucked up*. We were genuinely learning "the rules" of Rust programming on the fly, and that meant we made lots of mistakes. I found those mistakes very educational, and frequently used them to teach the limits of the language. [The chapter on leaking epitomizes this approach](https://doc.rust-lang.org/nightly/nomicon/leaking.html). The Rustonomicon is *the faultlore of designing Rust*, a tome of lessons taught through failures and mistakes.

An incomplete tome full of its own mistakes, because there are always more mistakes to be made and more lessons to be learned.

Because when things seem to work, you learn nothing. Are they working the way you expect? Are they working for the right reason? Will they keep working tomorrow? Did you just get lucky? Hell, you probably aren't even thinking about it at all anymore.

But the failures. Oh the failures. The failures contain multitudes.

Look at all these succulent and appealing things we wished to do but could not! Gaze upon our failures and become enlightened!

As I wrote more of the rustonomicon, I found this concept of learning through failure completely intoxicating. At the same time, I kept seeing people picking up Rust, trying to write a linked list, and failing miserably.

This failure was completely obvious to me, the person who learned all of the forbidden secrets of the language by working on data structures, but it was also really fucking annoying. Who the fuck picks up a language and goes "ah, first thing's first, time to write a linked list"?

*I* was reasonable and learned Rust by looking at binary trees, which epitomize elegance and ownership itself. (Which is of course why I got them removed from the standard library; they were too perfect for our horrible world.)

You know what, fuck it.

Let's write linked lists.

I'll make you [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/). Smoke the whole carton of linked lists, and see how you feel about them afterwards.

Oh no, I won't just tell you how to make them correctly! I will become the very embodiment of your naivety and innocence. I will create a whole book that consists entirely of compile error faceplanting. It will be my greatest shitpost.

What do you *mean* you're recommending this to others as an introductory text for the language?

Yes, somehow I got so mad about linked lists that I accidentally created the magnum opus of my budding philosophy. A book of pure faultlore, where all the lessons come from failure. A book where I scream and sob at the slightest hindrance so that you can find catharsis in our inevitable success.

A book where budding rustaceans learn to love and embrace compiler errors. Where they learn that the compiler isn't their enemy preventing progress, but their friend who helps them avoid mistakes.

Take joy in your failures and weave them into celebrations and traditions. Paint their portraits, sing their songs, and dance their beats.

Turn your faults into faultlore and share them with the world.


