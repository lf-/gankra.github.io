% Why Hashbrown Does A Double-Lookup

<span class="author">Alexis Beingessner</span>

<span class="date">March 20th, 2019 -- Rust Nightly 1.35.0</span>



I recently finished a detailed review of [hashbrown](https://github.com/Amanieu/hashbrown), which will likely become the new implementation for rust's std::collections::HashMap. One of the
most surprising things I found was in the implementation of `insert`, which was essentially:

```rust
fn insert(&mut self, key: K, val: V) {
    let hash = self.hash(&key);
    if let Some(old_val) = self.get_mut(hash, &key) {
        *old_val = val;
    } else {
        self.really_insert(hash, key, val);
    }
}
```

This shocked me because it was doing something that was so offensive to people who care about collection performance that we had [designed an entire API to help people avoid it](https://doc.rust-lang.org/std/collections/index.html#entries): it did two lookups in the map.

However, after some more discussion and review, I concluded that this implementation was reasonable. This post will try to cover why that is.





# Hashmap Basics

Before we get into the details, we need a quick primer on how hashmaps work.

A hashmap is basically just an array, and a way to convert keys to a seemingly-random index in that array (a hash function). We won't be talking about hash functions, so just assume we've solved that problem well.

Given this, doing insertions and lookups is conceptually very simple:

```rust
fn insert(&mut self, key: K, val: V) {
    let index = self.hash(key) % self.vec.len();
    self.vec[index] = Some((key, val));
}

fn get(&mut self, key: &K) -> Option<&V> {
    let index = self.hash(key) % self.vec.len();
    self.vec[index].as_ref().map(|pair| &pair.1);
}
```

Wow easy, can't be more effecient than that, hashmaps solved forever.

Not quite. Here's the issue: two different keys can be assigned the same index in our array, and we need to handle that in a way that lets both keys exist in our map. How we choose to solve this problem is basically the entire problem with implementing a HashMap.

A very simple solution is as follows: just walk along the array until we find an empty index (or matching key to overwrite). This approach of just trying to find a new index is called "open addressing", which will be the focus of our discussion (the most common alternative being an approach called "chaining").

Here's a simple implementation of open addressing that literally just iterates through the array indices until it finds an empty slot (this is known as first-come-first-serve linear probing, and is usually a bad strategy for complicated reasons we don't care about here):

```rust
fn insert(&mut self, key: K, val: V) {
    let mut index = self.hash(key) % self.vec.len();
    while self.vec[index].is_some() && self.vec[index].unwrap().0 != key {
        index = (index + 1) % self.vec.len();
    }
    self.vec[index] = Some((key, val));
}
```

However now that we're using open addressing, we have another problem: how to remove things. If you just delete an entry where you found it, this will corrupt the map. For instance, say we have 3 keys, A, B, and C, which all wanted to go in the same index:

```text
[..., A, B, C, _, ...]
      ^
      |
      where all 3 keys wanted to be inserted (A came first, then B, then C)
```

If we delete B naively, then C will become "lost":

```text
[..., A, _, C, _, ...]
      ^  ^
      |  |
      |  ...and gives up here, because we found an empty index
      |
      search for C starts here...
```

There are two solutions to this problem: backshifting, and tombstones.

With backshifting, if we see that the element after B would want to be in B's spot, we move it there (and possibly repeat this process over and over until we hit an empty spot):


```text
[..., A, C, _, _, ...]
```

With tombstones, if we see that there's an element after B, we mark the entry as "deleted" ("leaving a tombstone in its place"):

```text
[..., A, *, C, _, ...]
```

Tombstones are like free spots for insertion, but they let our search algorithm know that there used to be something there, so we should keep searching past that spot in case it displaced the key we're looking for.

Whether tombstones or backshifting are better for you depends on your implementation and workload. All of the implementations Rust's standard library has used over the years have been backshift based. Hashbrown is interesting because it bucks this trend and uses tombstones. So we'll be focusing on tombstones from here on out.






# Hashbrown and Tombstones

Tombstones lead to an interesting property of our insertion algorithm: the place that you prove that the key isn't in the table can be incredibly far from where you perform the actual insertion:


```text
[A, B, C, *, *, *, D, *, *, E, _]
 ^        ^                    ^
 |        |                    |
 |        where to insert      where you prove that the key
 |                             isn't in the table
 where you start
```

This can lead you to two different implementation strategies: the double simple loop, and the single complex loop (these implementations fleshed out to be a bit more implementation agnostic):


```rust
// The double simple loop
fn insert(&mut self, key: K, val: V) {
    let index = self.hash(key);

    for bucket in self.iterate_buckets_from(index) {
        if bucket.has_key() {
            if bucket.key() == key {
                // found the key, overwrite it
                bucket.set(key, val);
                return;
            }
            // keep searching
        } else if bucket.is_empty() {
            // the key isn't in here, now let's find where to insert
            break;
        }
        // keep searching
    }

    // Ok we know the key isn't in here, now *really* find where to insert it
    for bucket in self.iterate_buckets_from(index) {
        if !bucket.has_key() {
            // insert into the first non-key bucket (empty or tombstone)
            bucket.set(key, val);
        }
    }
}
```



```rust
// The single complex loop
fn insert(&mut self, key: K, val: V) {
    let index = self.hash(key);

    let first_tombstone = None;

    for bucket in self.iterate_buckets_from(index) {
        if bucket.has_key() {
            if bucket.key() == key {
                // found the key, overwrite it
                bucket.set(key, val);
                return;
            }
            // keep searching
        } else if bucket.is_tombstone() {
            if first_tombstone.is_none() {
                // Remember our first tombstone
                first_tombstone = Some(bucket);
            }
            // keep searching
        } else if let Some(target_bucket) = first_tombstone {
            // found empty, but walked over tombstone: overwrite first tombstone
            target_bucket.set(key, val);
            return;
        } else {
            // found empty, and never saw tombstone: overwrite empty
            bucket.set(key, val);
            return;
        }
    }
}
```

Knowing nothing else about our implementation, I would tend to favour the "single complex loop" approach, because it seems like it would do less work under heavy loads. Definitely something worth testing and benchmarking, though!

But hey, that double simple loop looks familiar, what was that implementation that was troubling us again?

```rust
fn insert(&mut self, key: K, val: V) {
    let hash = self.hash(&key);
    if let Some(old_val) = self.get_mut(hash, &key) {
        *old_val = val;
    } else {
        self.really_insert(hash, key, val);
    }
}
```

Hey that's the same thing!

So now we have a sense for why we might come to such an implementation, but why does it seem to be the *better* one for hashbrown? Well, hashbrown's big idea is to be optimized for SIMD. Which is to say: hashbrown is optimized to churn through buckets in our map in big chunks all at once. For instance, when compiled with sse2 support it can search 16 buckets at once. The odds that we take more than 16 elements to find where to insert is quite low.

And so our "two loops" are usually just "do two simple SIMD operations". Not quite so offensive when you say it like that!

Here's a sketch of the implementation in hashbrown:


```rust
fn insert(&mut self, key: K, val: V) {
    let hash = self.hash(key);

    // Trick to quickly reject almost every key without looking at it
    let hash_fragment = top7_bits_of(hash);

    for bucket_cluster in self.iterate_buckets_from(hash) {
        // SIMD-equals to get a bitvector of TRUE/FALSE
        // We iterate this by repeatedly grabbing and clearing the lowest bit,
        // and stop when the bitvector is 0
        for bucket in bucket_cluster.get_all_matches(hash_fragment).iter_bits() {
            if bucket.key() == key {
                bucket.set(hash_fragment, key, val);
                return;
            }
        }

        // SIMD-equals to get a bitvector, and just check if it's 0
        // (if it is, this cluster is entirely keys and tombstones, so keep going)
        if bucket_cluster.get_all_matches(EMPTY) != 0 {
            // EXTREMELY likely that we do this on the first iteration
            break;
        }
    }

    for bucket_cluster in self.iterate_buckets_from(hash) {
        // SIMD collect the top bits of each byte into a bitvector
        // (both empty and tombstone buckets have the high bit set)
        let bitvec = bucket_cluster.get_all_empty_or_tombstone();
        if bitvec != 0 {
            // EXTREMELY likely that we do this on the first iteration
            let idx = bitvec.lowest_set_bit();
            bucket_cluster.get_bucket(idx).set(hash_fragment, key, val);
            return;
        }
    }
}
```

And that's why insertion in hashbrown does a double-lookup.

