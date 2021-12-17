% Building a HashMap in Rust - Part 1: What's a HashMap?

## Aria Beingessner

This is the fourth entry in a series on implementing collections in the Rust programming language. The full list of entries can be found [here][index].

In contrast to the [previous][previous] post, which dove into Rust's BTreeMap and the idea of B-Trees in general, this post will be split up into two posts. In this post we will take a look at the general problems faced by hashtables, as well as the robin hood hashtable scheme. In the next post, we will take a high-to-mid-level look at Rust's standard HashMap implementation (surprise, it's a robin hood hashtable), and what it does to be safe and fast.

## What's a Hash?

Before I begin, I would just like to take a moment to reflect upon a formative quote offered up by my supervisor on the topic of hashing.

> Fuck your fucking hashtables!
>
> \- Dr. Michiel H. M. Smid

From this point on we will refer to hashtables as hashmaps, and robin hood as robinhood. For details, see the above quote.

------

There are two god-kings of the practical data structuring world: array stacks and hashmaps. At a high level, array stacks are really simple, and all basically the same. Hashmaps, on the other hand, are surprisingly complex and diverse. However at the highest level hashmaps all basically have the following structure:

* Make a big array
* Get a magic function that converts elements into integers (hashing)
* Store elements in the array at the index specified by their magic integer
* Do something interesting if two elements get the same index (hash conflict)

In doing this, it is generally expected that one is able to insert, remove, and lookup keys in a hashmap in O(1) time. Literally as fast as theoretically possible! Unfortunately, in practice this O(1) is quite elusive. Or, when obtained, comes at the cost of a very *high* constant cost per operation. 

## Handling Collisions

Hashing is a tricky problem in-and-of-itself, but for now we will assume we have solved the problem. This is because the problem of hashing is not very interesting until we understand how collisions are handled. So for now we will assume that, given an array of length n, every element deterministically gets mapped to an apparently random integer in the range [0, n) with uniform probability.

Even with this miraculous assumption, it is still expected for collisions to occur. Especially as the map gets more full. Just *losing* one of the two elements is, presumably, not something many people want to happen. So we need some way to deal with this happening. There are two major families of collision resolution strategies: chaining, and open addressing.

Chaining is the strategy you are most likely to see in an undergraduate data structure class, and definitely the easiest to implement and reason about. Basically, every index in your array doesn't store *elements*, but rather *another* collection. Typically, the collection described is a linked-list. Every time an element is inserted into a particular index, we just add it to the list. To find if an element is in the map, we jump to its index, and then do a linear search of the list at that index. To remove an element from the map, we remove it from its list. That's it.

If we have a perfect hash function, and our hash table contains n elements, it's easy to see that such a hashmap will take O(1) expected time to do any particular operation. Each list will contain about one element on average, meaning it can be searched in O(1) time. Some lists will contain more elements, and others will contain less, but on balance it should all work out.

Note that the choice of a linked list is fairly arbitrary. One could use an array stack or binary search tree. I'm not clear why a linked list is the most popular choice here.

Regardless, chaining isn't exactly the most efficient choice. It necessarily involves *some* level of indirection to store the auxiliary collections in an array, and those collections are going to require additional allocations to grow to fit an arbitrary number of elements. This means more wasted space, and a higher constant cost per operation.

This is where open addressing comes in. Rather than introducing any of this noise about auxiliary structures, it asks the question "why don't you just look somewhere else"? Given some deterministic way of searching from the location of the conflict for different empty spaces, we can avoid adding indirection or extra allocations to our hashmap. However, this complicates the analysis. Now we don't just need to worry about a *true* hash collision, we also need to worry about collisions with displaced elements. A poor choice of algorithm here can completely subvert even a *perfect* hash function!

However before we dig into open addressing schemes, let's look at the problem of actually getting good hashing.

## Hashing

Hashing is *technically* an orthogonal issue to hashmap implementation. Any decent hashing scheme can be plugged into any decent hashmap implementation. But it's an important problem that can influence hashmap design decisions. Hashing for a hashmap is also a fairly different problem from e.g. hashing a password. As such, it's worth discussing here.

The goal of hashing is to take arbitrarily complex data and reduce it to a single (seemingly random) number. This number will be used to decide where the data should be inserted. Unfortunately, as we saw in the previous section, collisions can and will happen. The more collisions a function produces, the worse the map's performance becomes.

In fact, this is actually an attack vector on applications! The attack is variously known as a *denial of service attack*, *algorithmic complexity attacks*, or *hash flooding attack*. If a malicious user can feed carefully crafted data to a hashmap that all happens to hash to the same value, the hashmap will often experience disastrous performance. 

Consider a hashmap using linked-list chaining. If n distinct elements can be fed to this map that all hash to 0, then the map will effectively degenerate into a *linked list*. Worse, the entire list will be searched on every insertion. Simply constructing the map will take O(n<sup>2</sup>) time!

These kind of attacks were first described in detail in 2003 by Crosby and Wallach [(html)][perl-dos-html] [(pdf)][perl-dos-pdf] when they demonstrated attacks against the Perl programming language and several applications. Klink and Wälde popularized this kind of attack in 2011 [(video)][web-dos-vid] [(slides)][web-dos-slides] when they published an attack against the hashmap implementations in PHP, Java, Ruby, and several other languages that was able to lock up some popular web frameworks using relatively small POST requests. 

That is, they literally just sent a few POST requests with carefully crafted argument names and the application completely locked up. They didn't overflow any buffers or exploit any bad sanitation logic. Simply the act of storing the POST arguments in a hashmap was too much for them to computationally handle. That's pretty scary, if you ask me!

The solution to this specific attack is simple and well-known: use a *random* hash function. If you have a random hash function, then it should be impossible for an attacker to determine a set of keys that all hash to the same value. The attacks by Crosby and Wallach were largely against fixed hash functions, for which collision families could be pre-computed by brute-force or slightly more advanced techniques. 

Generally, this is done by using a single hashing algorithm that incorporates a secret key in the hashing process. A random hash function is chosen by generating a random key per process, thread, or hash table instance. With a sufficiently large key, this should in theory prevent an attacker from identifying a sequence of values that hash to the same number. A collision in one function is unlikely to be a collision in another random function. 

However some care must be taken, as a sufficiently motivated attacker may be able to identify which function was chosen by probing the application's behaviour, or exploiting flaws in the algorithm itself. In 2012, attacks against MurmurHash and CityHash were presented by Aumasson, Bernstein, and Boßlet [(video)][murmur-dos-vid] [(slides)][murmur-dos-slides]. This is in spite of the fact that these hashing algorithms *were* random.

To my knowledge, [SipHash 2-4][siphash] (which was released along with announcement of the MurmurHash exploit) is the current en-vogue solution to this problem. It is a reasonably performant random hash function with a decent amount of cryptographic analysis applied to it. However it's not as fast as some of the "insecure" hash functions. As such one is forced to make a choice: either you can be as fast as possible, or you can be as secure as possible. Because this is a relatively obscure attack, I recommend that one defaults to the *secure* choice, with the option of a fast choice (not all data is worth worrying about). I make this recommendation based on a simple cost-benefit analysis:

* If the user is aware of this attack and the details of their application, they can always override any default and get the "right" choice.
* If the user is *unaware*, and the default is *secure*, then at worst their application runs a bit slow (and maybe you get some bad benchmark publicity).
* If the user is *unaware*, and the default is *fast*, then at worst their application gets DDOS'd and they lose $$$.

For reference, this is the position that Rust's standard implementation takes (more on that next time). 

As an aside, SipHash is a *streaming* hasher. That means it consumes an arbitrary number of bytes until you tell it to spit out an answer. This is actually *super* convenient for a hashing API. Need to write a Hash implementation for a composite type? Just feed the hasher all the parts individually. Need to hash an arbitrary-length string? Just hash each byte of the string.

The *alternative* to a streaming hasher is hash coding. Hash coding is super annoying and problematic. It requires the implementer of Hash to somehow reduce their type to a single integer which the hasher then hashes. This adds another place for hashmaps to be attacked, and it's completely out of the hashmap implementor's control. Classic mistakes like implementing hashcoding as the xor of the elements leads to easy exploits (x xor x = 0 for all x).

## Robinhood Hashmaps

Alright, let's get back to open addressing. There are a few different strategies here, varying from simple to exotic. As cool as [cuckoo hashing][cuckoo] is, I will not be covering that. Instead we're going to focus on the simple strategies.

So let's say we've encountered a hash collision. The index we want to occupy is full. What's the simplest strategy you can possibly think of, to find an empty space?

Walk to the right.

Yep, we just keep walking to the right (with wrapping) until we find an empty space, and then use that. This is called *linear probing*. 

To search for an element, we jump to its index, and then linear search the array until we find a match or an empty space. 

To remove an element, we pop it out and backshift all the elements that had to walk over it. We'll call this the *first come first serve* (FCFS) strategy.

This sounds like a pretty promising idea. CPUs love linear access patterns, right?

An alternative strategy is to instead *evict* any element we find and tell *them* to search for a different space. Search and removal is otherwise the same. We'll call this *last come first serve* (LCFS). This definitely produces a *different* distribution, but it's not clearly better. It also involves a lot more writes.

I hope you can see that these strategies can get pretty degenerate. There's no obvious bound on how far we'll need to walk before we find an empty space. In fact, here's a simple example that demonstrates this degenerate behaviour:

Consider inserting a sequence of elements `a, b, c, d, e, f` that hash to the values `0, 0, 1, 2, 1, 0` respectively.

with FCFS:

```plain
[           ] array initial
[a          ] a inserted
[a b        ] b inserted (a displaces b)
[a b c      ] c inserted (b displaces c)
[a b c d    ] d inserted (c displaces d)
[a b c d e  ] e inserted (b, c, d displaces e)
[a b c d e f] f inserted (everyone displaces f)
[0 1 1 1 3 5] final distances from ideal
```

Here we can see that `f` has to travel a full distance of 5.

With LCFS:

```plain
[           ] array initial
[a          ] a inserted
[b a        ] b inserted (b displaces a)
[b c a      ] c inserted (c displaces a)
[b c d a    ] d inserted (d displaces a)
[b e c d a  ] e inserted (e displaces c, d, a)
[f b e c d a] f inserted (f displaces everyone)
[0 1 1 2 2 5] final distances from ideal
```

Here we can see that `a` has to travel a full distance of 5.

The final distributions are quite different, but the end result is the same: one element is maximally displaced from its ideal location.

*Ideally* this won't be a problem. On average everyone is pretty close to their ideal location. However if a *single* element is very far from its ideal location, then this still opens up the potential for a denial of service attack. Inserting that one element will still take a disproportionately long amount of time. If the attacker has some way to trigger searches, then searching for this element will also be bad.

So for our purposes we aren't so much interested in the *average* distance traveled, as we are the *variance*. Enter robinhood hashing.

This scheme actually has *several* crazy variants, as it was the subject of Pedro Celis' PhD thesis [(pdf)][thesis]. However almost all of the variants are stupid in practice. We will focus on a particular variant that is *basically* the same as the two schemes we just looked at. Like FCFS, when we find a collision, we linearly probe for an empty space until we find one. However, if we ever find an element that isn't as far from its ideal as we currently are, then we take its place and have *it* continue the search, just like LCFS. 

In other words, robinhood takes from the rich, and gives to the poor.

This seemingly simple modification to the FCFS and LCFS strategies is a *huge* improvement from a variance perspective. While FCFS and LCFS easily permit a single element to get pushed really far, robinhood requires that for an element to be pushed some distance d, it must have been displaced by an element that traveled at least d-1. Which in turn must have been displaced by a d-2 element, and so on. Let's try out our favourite example on robinhood -- `a, b, c, d, e, f` that hash to the values `0, 0, 1, 2, 1, 0` respectively:

```plain
[           ] array initial
[a          ] a inserted
[a b        ] b inserted (a displaces b because it was first)
[a b c      ] c inserted (b displaces c because c is farther)
[a b c d    ] d inserted (c displaces d because d is farther)
[a b c e d  ] e inserted (b displaces e (farther)
						, c displaces e (tie)
						, e displaces d (farther))
[a b f e c d] f inserted (a displaces f (tie)
						, b displaces f (tie)
						, f displaces c (farther)
						, e displaces c (tie)
						, c displaces d (farther))
[0 1 2 2 3 3] final distances from ideal
```

The *average* distance for all three examples is in fact the same, but the variance on robinhood is better. The farthest any element traveled was 3 steps. Although consequently less elements have an *excellent* search time. Everyone's search time is just "okay". Still, this means robinhood is a more reliable scheme. 

If you're ultra keen, I suppose about now you're violently vibrating at your desk over some omitted details. First, how do we *know* how far an element has traveled? Well, an element's hash tells us where it *wants* to be, and we definitely know where in the array we currently are. So it's just simple subtraction. 

But hashing is slow. Should we seriously be hashing every element we see in our search? You definitely *can* if space is your top priority, but a better idea is probably to just *store* an element's hash with it. This is a bit wasteful, but the space-time tradeoff is totally worth it. We'll see in the second part how this can be used to make search even *more* reliable and efficient.

Another issue: do robinhood hashmap lookups really take O(1) expected time? Even with our low variance, it's not obvious that we'll only walk a *constant* distance. This is a tricky question to answer. First off, it's sensitive to the *load factor* our map has. I can pretty confidently say that a hashmap with only 1% of its indices full is going to have O(1) search time, but what about 90%? 99%?

For this we can look to a paper by Luc Devroye and Pat Morin from 2003 [(pdf)][robin-bound]. I'll just quote their abstract:

> We hash ~ alpha*n elements into a table of size n where each probe is independent and uniformly distributed over the table, and alpha &lt; 1 is a constant. Let M be the maximum search time for any of the elements in the table. We show that with probability tending to one, M is in [log<sub>2</sub>log n + a, log<sub>2</sub>log n + b] for some constants *a* and *b* depending upon alpha only. This is an exponential improvement over the maximum search time in case of the standard FCFS collision strategy.

So the expected worst-case search time is O(loglog n) if we have at most some constant load factor. I'll admit, that's not O(1). But let's be serious here for a second: even if you somehow manage to populate a hash table with 2^64 elements, loglog 2^64 is... 6. For all intents and purposes, we've got constant-time searches.

As a final little tidbit for why robinhood kicks ass, consider searching for an element in the table. Let's say we've walked d steps, and we encounter an element that is some distance *less* than d from *its* ideal. If the element we're searching for *was* in the table, then it would have displaced this one, right? So, we can stop. We don't need to bottom out into an actually empty space to give up on the search like FCFS or LCFS, we can stop early! Nice.

## Conclusions

Hashmaps and hash functions enjoy a diverse ecosystem of literature and implementations. For performance and simplicity, I recommend a robinhood hashmap, which enjoys O(1) expected time per query without the need for chaining.

For security, I recommend SipHash 2-4 as a reasonable *default* hashing algorithm, with some way to override the default for performance.

Next time we'll dig into Rust's standard implementation of a HashMap using exactly the designs described above. Although theory time isn't *quite* over. Our description will feature a *novel* theoretical result for robin hood hashmaps!

See you next time!

[previous]: http://cglab.ca/~abeinges/blah/rust-btree-case/
[web-dos-slides]: http://events.ccc.de/congress/2011/Fahrplan/attachments/2007_28C3_Effective_DoS_on_web_application_platforms.pdf
[web-dos-vid]: https://www.youtube.com/watch?v=R2Cq3CLI6H8
[perl-dos-pdf]: http://www.rootsecure.net/content/downloads/pdf/dos_via_algorithmic_complexity_attack.pdf
[perl-dos-html]: https://www.usenix.org/legacy/events/sec03/tech/full_papers/crosby/crosby_html/index.html 
[murmur-dos-vid]: https://www.youtube.com/watch?v=wGYj8fhhUVA
[murmur-dos-slides]: https://131002.net/siphash/siphashdos_appsec12_slides.pdf
[siphash]: https://131002.net/siphash/
[cuckoo]: http://en.wikipedia.org/wiki/Cuckoo_hashing
[thesis]: https://cs.uwaterloo.ca/research/tr/1986/CS-86-14.pdf
[robin-bound]: http://cglab.ca/~morin/publications/hashing/robinhood-siamjc.pdf
[index]: http://cglab.ca/~abeinges/blah/
