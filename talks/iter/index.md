% OMG ITERATORS

Alexis Beingessner

Carleton University + Mozilla Research

[<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://cglab.ca/~abeinges)
[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://rust-lang.org)

rendered: http://cglab.ca/~abeinges/talks/iter/

raw: http://cglab.ca/~abeinges/talks/iter/index.md





# Let's talk about this for 30 minutes

```rust
pub trait Iterator { // I can yield things
    type Item;       // Things I yield
    // How I yield them
    fn next(&mut self) -> Option<Self::Item>;
}
```

* `mut` because we want to *progress*
* `Option` coalesces `has_next` and `get_next`
  * Some(item) - I have next, and here it is
  * None - I don't have next





# Usage

```rust
for i in 0..10 {
    println!("{}", i);
}
```

* `i`: name for `item` in `Some(item)`
* `0..10`: the thing to iterate (0 to 10, 10 excluded)



# Collection Iterator Dream Team

* IntoIter
* Iter
* IterMut

(these are structs)





# OMG SO MANY WHY??

Ownership!





# IntoIter - Owned Values (T)

* Moves the data out of the collection
* You get total ownership!
* Can do anything with data, including destroy it





# What if we only had IntoIter

```rust
fn process(data: Vec<String>) {
    for string in data.into_iter() {
        // I got all the data, it's mine!
        println!("{}", string);
    }

    // Oh no! Iterating consumed it :(
    println!("{}", data.len()); //~ERROR
}
```

Trash Language; Quit Forever





# Iter - Shared References (&T)

* Shares the data in the collection
* Read-only access
* Can have many readers at once





# Iter lets you look but not touch

```rust
fn print(data: &Vec<String>)
    for string in data.iter() {
        // All I can do is read :/
        println!("{}", string);
    }

    // Yay it lives!
    println!("{}", data.len());
}
```





# Everyone can look at the same time!

```rust
fn print_combos(data: &Vec<String>) {
    let len = data.len();
    for string in data.iter() {
        // Iterate and query at the same time!
        println!("{} {}",
                 string,
                 &data[random_idx(len)]);
    }
}
```



# IterMut - Mutable References (&mut T)

* Loans the data in the collection
* Read-Write access
* Only one loan at once




# IterMut gives you exclusive access

```rust
fn make_better(data: &mut Vec<String>) {
    for string in data.iter_mut() {
        // Ooh I can mutate you!
        string.push_str("!!!!!");
        // But I can't share :(
    }
}
```




# Drain - I Drink Your Milkshake (🍼T)

* Partially moves the data
* Full access to the elements
* Doesn't destroy the container





# Drain lets you partial move

```rust
let mut data = vec![0, 1, 2, 3, 4, 5];

for x in data.drain(2..4) {
    // got exclusive access so we can "drain"
    // the values out but leave the vec alive
    consume(x);
}

// data lives on! We can reuse the allocation!
// Bulk `remove`!
// Dat Perf. ;_;
assert_eq!(&*data, &[0, 1, 4, 5]);
```





# Recap

Iterators naturally fall out of ownership:

* IntoIter - `T`
* Iter - `&T`
* IterMut - `&mut T`
* Drain - `🍼T`






# IterMut is kinda weird...

Rust doesn't understand when indexing is disjoint:

```
let mut data = vec![1, 2, 3, 4];

let ptr1 = &mut data[0];
let ptr2 = &mut data[1]; //~ ERROR

*ptr1 += 5;
*ptr2 *= 3;
```

(I would argue this is a good thing)




# IterMut is kinda weird...

But it's ok with IterMut?!
Ownership busted?!

```
let mut data = vec![1, 2, 3, 4];
let mut iter = data.iter_mut();

let ptr1 = iter.next().unwrap();
let ptr2 = iter.next().unwrap();

*ptr1 += 5;
*ptr2 *= 3;
```





# IterMut 100% Legit and Safe

* Iterators are "one shot"
* Each element yielded *at most* once
* Can't get a fresh IterMut while refs live

(not true for indexing in general)





# Why does the API allow this?

```rust
impl<'a, T> Iterator for IterMut<'a, T> {
  type Item = &'a mut T;
  fn next<'b>(&'b mut self) -> Option<&'a mut T> {
    ...
  }
}
```

`'a` is not associated with `'b`, so `next` doesn't care if you still have a `'a`

*Shh... It's okay borrow checker, only dreams now*






# Crazy!

How can Rust let you implement this?!?






# The Borrow Checker is Crazy Smart

You can statically prove to the compiler that this works.





# IterMut for an Array

```rust
# use std::mem::replace;
pub struct IterMut<'a, T: 'a> { data: &'a mut[T] };

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        let d = replace(&mut self.data, &mut []);
        if d.is_empty() { return None; }

        let (l, r) = d.split_at_mut(1);
        self.data = r;
        l.get_mut(0)
    }
}
```


# Not Just Arrays

Same basic idea works on singly-linked lists and trees -- anything with clear
ownership that you can get subviews into.







# What if I don't *want* them to call next again?

Totally valid use-cases:

* Mutating collection structure during iteration
* Non-disjoint items (windows or repeats)
* Items constructed and stored in one place

None of these are sound because Iterator says
"you can always call next again".





# Borrow Checker Escape Hatch: Indices!

```rust
// Vec of enemies with health.
// When they reach 0 we "die!" and drop them.
// Concurrent modification and iteration!
for i in (0..enemies.len()).rev() {
    enemies[i] -= 1;
    if enemies[i] == 0 {
        println!("die!");
        enemies.swap_remove(i);
    }
}
```

You only wanted to work with arrays, right?






# More Robust: StreamingIterator

```rust
pub trait StreamingIterator {
  type Item;
  fn next<'a>(&'a mut self)
      -> Option<&'a mut Self::Item>;
}
```

*Now* the lifetimes are attached!

Statically prevented from calling `next` twice.





# Good

Actual mutual exclusion!

```
let ptr1 = iter.next().unwrap();
let ptr2 = iter.next().unwrap(); //~ERROR
```





# Bad

`&mut Item` is hardcoded.

* `&Item`?
* `Item<'a>`?





# We can't express this generically today

 `(╯°□°)╯︵ ┻━┻`

(need higher-kinded-whatzits)




# Going Backwards!

```rust
pub trait Seeker {
  fn next(&mut self) -> Option<&mut Self::Item>;
  fn prev(&mut self) -> Option<&mut Self::Item>;
}
```





# Going Crazy!

```rust
pub trait Cursor {
  fn next(&mut self) -> Option<&mut Self::Item>;
  fn prev(&mut self) -> Option<&mut Self::Item>;
  fn insert(&mut self, Self::Item);
  fn remove(&mut self) -> Option<Self::Item>;
}
```





# We don't need no stinkin' traits!

* Can't be generic over collections in Rust *anyway*
* Can't really use Cursor in a for-loop *anyway* (the loop var would
  borrow the iterator for the whole loop body)

http://contain-rs.github.io/linked-list/linked_list/struct.Cursor.html





# Conclusions

* Iter, IterMut, IntoIter, and Drain "natural" reflections of ownership
* Iterator always lets you call `next` again
* StreamingIterator prevents you calling `next`
* Enables `prev`, `insert`, `remove`
* Works concretely today






