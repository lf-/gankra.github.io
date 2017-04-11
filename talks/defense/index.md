% You Can't Spell Trust Without Rust

<center><h1>Alexis Beingessner</h1></center>

<center>[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://rust-lang.org)</center>






# The Actual Thesis

By enabling programmers to specify where and when data can be mutated, and where
and when it lives, ownership enables programmers to design interfaces in terms of trust.
(and this is useful and pleasant!)








# Problems

* use-after-free
* index out of bounds
* iterator invalidation
* data races
* memory leaks





# Trust

Common theme: code trusting data

*What's the justification?*





# Naivety

Blind trust

* Good for control
* Bad for safety
* Mixed ergonomics




# Paranoia

Static guarantees - omissions or static analysis

* Good for safety
* Bad for control
* Mixed ergonomics




# Suspicion

Verify at runtime

* Good for ergonomics
* Ok for safety
* Ok for control






# Ownership

Emergent from 3 systems

* Affine Types
* Borrows and Regions (lifetimes)
* Privacy





# Affine Types

* "usable at most once"
* Solves use-after (without pointers)
* Solves sequencing






# Borrows and Regions

* Pointers tied to regions of code they can't escape
* Mutating through pointer needs unique access (mut ^ share)
* Solves use-after (with pointers!)
* Solves iterator invalidation
* Solves data races






# Privacy

* Makes it possible to build safe abstractions
* Language is extensible with naive interfaces (Unsafe Rust)






# Case Study: Indexing

Can ergonomically express all kinds of trust

* Naive: `get_unchecked`
* Suspicious: `get` and `[]`
* Paranoid: iterators






# Case Study: Entry API

Demonstrates external vs internal interfaces

* Internal: client executes in your context (always safe)
* External: you execute in client's context (safe due to ownership!)





# Generativity

Can bootstrap fancy type-system stuff on top of region analysis






# Limitations - Cycles

Rust likes *unique* ownership.





# Limitations - Leaks

* Largely handled by destructors
* Destructors can be leaked!
* Can't rely on destructors for safety
* But this is a library decision!




# Case Study - Scoped Threads

* Brings it all together
* Mutating data on parent thread's stack from child threads
* No unnecessary synchronization
* 100% safe





