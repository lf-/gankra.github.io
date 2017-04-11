% The Dark Arts of Unsafe Rust Programming

Alexis Beingessner

Carleton University + Mozilla Research

[<img src="icon.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://cglab.ca/~abeinges)
[<img src="rust.png" width="250" style="display:inline; box-shadow:none;"></img>]
(http://rust-lang.org)

rendered: http://cglab.ca/~abeinges/talks/moz/

raw: http://cglab.ca/~abeinges/talks/moz/index.md



# The Rustonomicon

[doc.rust-lang.org/nightly/nomicon/](https://doc.rust-lang.org/nightly/nomicon/)

It's a book. I wrote it. That's what I did this summer.

Presentation over.

(Actually I wrote two, but I was *supposed* to write the first)

[cglab.ca/~abeinges/blah/too-many-lists/book/](http://cglab.ca/~abeinges/blah/too-many-lists/book/)



# What's a Safe Language?

Memory Safety and Type Safety, mostly

Safe: Python, Ruby, Javascript, Java...

Unsafe: C, C++





# Everyone is Unsafe Because of FFI

C is the lingua franca

Everyone has to support C FFI

Safe Language + FFI = "lol safety"





# Rust is Safe

* No indexing out of bounds
* No dangling pointers / use-after-free
* No null pointers (!)
* No iterator invalidation (!!)
* No data races (without pervasive atomics!!!)




# Rust is Helpful

* Hard to leak memory/destructors
* Hard to see exception-safety issues (!)
* Hard to do concurrency wrong (!!!)





# Rust is Unsafe?

Rust is two languages: Safe Rust and Unsafe Rust

Safe Rust is safe... except it can FFI into Unsafe Rust

(So it's as safe as any language ever is)





# Unsafe Rust

Exactly the same as Safe Rust -- but more things are allowed:

* Dereference raw (C-like) pointers
* Call unsafe functions (including C functions, intrinsics, and the raw allocator)
* Implement unsafe traits
* Mutate statics

*That's it*





# Why is this Ok?

All FFI into Unsafe Rust must be demarcated with the `unsafe` keyword.

You can `#[deny(unsafe)]` at the project or module level

Rust community takes pride in not using `unsafe` -- don't want to go back to C





# Modularity of Unsafety

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx < arr.len() {
        unsafe { Some(*arr.get_unchecked(idx)) }
    } else {
        None
    }
}
```

No need to worry about `arr` being null, dangling, uninit

Only need to worry about indexing out of bounds




# Modularity of Unsafety

```rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx <= arr.len() {
        unsafe { Some(*arr.get_unchecked(idx)) }
    } else {
        None
    }
}
```

Oops! Safe code caused UB!

`unsafe` taints the surrounding code.




# Modularity of Unsafety

```rust
pub fn push(&mut self, elem: T) {
    if self.len == self.cap { self.reallocate(); }
    unsafe {
      ptr::write(self.ptr.offset(self.len), elem);
      self.len += 1;
    }
}
fn make_room(&mut self) {
    // "grow" the capacity (unsound)
    self.cap += 1;
}
```




# Modularity of Unsafety

make_room isn't a problem because *privacy*

Only Vec's module can access make_room

`unsafe` taint is limited to module boundaries

Public API is safe -- unsafety captured!






# Conclusions

Rust is awesome, safe, and helpful

Unsafe Rust gives an FFI-like escape hatch

Much Better than FFI into C

Unsafety is easily contained by privacy
