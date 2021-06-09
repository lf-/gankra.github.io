% The Apple Compact Unwinding Format: Documented and Explained

<header>
    <p class="author">Alexis Beingessner</p>
    <p class="date">June 9th, 2021</p>
</header>

A few years ago, Apple introduced a new kind of unwinding info (I'll explain what
that is in the next section) -- the "compact unwinding format", which can be found in
`__unwind_info` sections of binaries.

Clang also started preferentially emitting that format over DWARF CFI on Apple's
platforms. So to generate good backtraces on Apple platforms, you need to be
able to parse and interpret compact unwinding tables.

Unfortunately, this format is defined only by its implementation in llvm. Notably
these files include lots of useful details, but you need to do a lot of studying
of the actual implementation to figure everything out:

* [Header describing layout of the format](https://github.com/llvm/llvm-project/blob/main/libunwind/include/mach-o/compact_unwind_encoding.h)
* [Implementation that outputs the format](https://github.com/llvm/llvm-project/blob/main/lld/MachO/UnwindInfoSection.cpp)
* [Implementation of lldb interpreting that format (CreateUnwindPlan_x86_64 especially useful)](https://github.com/llvm/llvm-project/blob/main/lldb/source/Symbol/CompactUnwindInfo.cpp)

Firefox's crash reporter needs good backtraces on Apple platforms, so learning
this format and implementing a parser/interpreter for it became my problem.
As I often do, I proceeded to write complete documentation for the format,
to the best of my ability.

Note however that my work omits details on the "personality" and "LSDA" parts of the format,
as those are only needed for running things like destructors, which aren't needed
for backtraces.

This article is an expanded version of the [enormous doc-comment I wrote for my
implementation](https://github.com/getsentry/symbolic/blob/49bd892c2efbfcf502d20164dac942101ddb43cb/symbolic-debuginfo/src/macho/compact.rs) in a soon-to-be-released version of [symbolic-debuginfo](https://github.com/getsentry/symbolic/) (which may eventually be moved into [goblin](https://github.com/m4b/goblin)).

My implementation and documentation is based on llvm commit `d480f968ad8b56d3ee4a6b6df5532d485b0ad01e`
and observations of MacOS around April-June 2021.

Also: A BIG thank you to the various folks who helped me debug/test/review/investigate this format.
While I wrote all the code and documentation, their input was invaluable in getting it
the work done properly.


# Background: Unwinding and Debug Info

(This section can be skipped entirely if you feel like you understand unwinding.)

So you have a running program. Functions call other functions, so you need a
mechanism to save the caller's state while the callee runs, and then restore
that state when the callee completes. Some of this is defined by
[ABIs](https://gankra.github.io/blah/rust-layouts-and-abis/), but even with a
defined ABI there's a lot of wiggle room for the callee to mess around.

Say for instance that your ABI mandates that the caller's `rbx` register must
be preserved by the callee. At one extreme, the callee can just not *use* `rbx`
but that's usually pretty impractical. At the other extreme, the callee can save
(push) `rbx` to the stack and restore (pop) it when it returns. Then
the callee can use `rbx` as much as it pleases, at the cost of always needing
to push and pop it.

In between these two extremes you find many Clever Tricks:

* Only save `rbx` in paths that need to modify it
* If `rbx`'s value is incidentally copied into another register, restore it from that register
* Oh on this path `rax` holds `rbx + 5`, so we can restore `rbx := rax - 5`
* etc, etc, etc

This is generally all Good Fun for an optimizer since these details are private
to the implementation of the callee -- it can do whatever it wants as long as it
figures out how to restore the caller's registers for every `return` in the
function.

Except backtraces and unwinding exist.



## Unwinding

Whenever you decided to unwind (panic or throw an exception) or generate a backtrace
(for a crash, step debugger, or a profiler trace), you are faced with the following problem:

You know the current address in the binary your program is executing (`rip`/`pc`)
and the current stack pointer (`rsp`/`sp`). Now figure out:

1. (optional) What function you're executing (and ideally what line)
2. What the return address is (the address the caller was last executing)
3. How to restore the caller's registers (most importantly the stack pointer)

And then repeat this over and over until you have walked over the whole stack.




## Standard Prologues (Frame Pointer Unwinding)

Steps 2 and 3 can be minimally done very easily if all functions uses a standard
"prologue". On x64, the standard prologue uses frame pointers, and is as follows:

1. caller performs a CALL (implicitly PUSHes the return address (`rip`) to the stack)
2. callee PUSHes `rbp` (the frame caller's frame pointer -- where it's stack starts)
3. callee sets `rbp := rsp` (initializes its own frame pointer)

If you know a function uses this calling convention, you can minimally complete
Unwinding Steps 2 and 3 (only restoring `rip` and `rsp`) with the following:

```
rsp := rbp - 16
rip := *(rbp - 16)
rbp := *(rbp - 8)
```

or natively:

```
mov %rsp %rbp
pop %rbp
ret
```

This convention kicks ass -- it gives you perfect super fast backtraces at the
cost of:

* ~2 extra instructions in your prologue
* ~2 extra instructions in your epilogue (you probably need to restore `rsp` anyway)
* permanently reserving a register to hold the frame pointer (`rbp`/`fp`)

If you want to do Unwinding Step 3 properly and restore all the callee-save
registers, you can also push all the callee-save registers in the
prologue as well.

Good news: [Apple Mandates Frame Pointers on ARM64](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms).

Bad news: No one else does, and optimizers gonna optimize. Especially on x86 where
register pressure is really high and `ebp` is so juicy and tempting.




## Debug Info

At this point you need to know what debug info is to do Unwinding Step 1, and do harder versions
of Unwinding Steps 2 and 3.

Debug info is kind of a blanket term for "extra metadata about a binary". There are
many kinds of debug info and container formats. The 3 most notable binary container
formats are:

* PE32(+) (Microsoft)
* Mach-O (Apple)
* ELF (Everyone Else)

Given the subject of this article, I will only really be focusing on Mach-O,
but *a lot* of this will also apply to ELF, as they both heavily use DWARF.

These container formats are generally broken up into sections -- things like constants (`.rodata`),
instructions to execute (`.text`), and well, debug info (`.debug_info`). Apple likes to prefix
these with two underscores, so I may interchangeably refer to `.__eh_frame` or `.eh_frame` --
they're the same.

The most common debug info format you will encounter is DWARF, which is actually
more of a family of formats for the various kinds of debug info sections. If you're
dealing with any kind of Unix/Linux/BSD variant you're gonna get a lot of DWARF,
and Apple's platforms are no exception.

For our purposes, the most notable DWARF sections are:

* `.debug_info` -- contains tables mapping addresses in the binary to source function names / lines
* `.debug_frame` -- contains unwinding tables(!!)
* `.eh_frame` -- same as debug_frame but non-standard/tweaked

To properly use these sections you need to know one more detail: an executable
is actually many disparate relocatable pieces of code that have been stitched together
by both a static linker (at compile time) and a dynamic linker (at runtime). To
handle this, the executable is sliced up into "modules" which generally refer to
a particular static or dynamic library (e.g. `libsystem_pthread.dylib`).

So given a particular instruction address, you need to lookup what module has
been mapped to that address, and then *within* that module you need to lookup
its `.debug_info`/`.eh_frame` sections. All addresses those sections refer to
will be relative to the start of that module, so you will need to add/subtract
that base address in various places to map back to the actual running executable.

I've only ever worked at the point where the module lookup has already been resolved,
so I'll leave that as an exercise to the reader.

With all that said, Unwinding Step 1 is simple:

Lookup the instruction address in the `.debug_info` section. Boom you have
function names and lines. That's it. That format isn't the focus of this.
There's plenty of DWARF parsers/interpreters, and the official specification
is quite good.

Of more interest to this article is `.debug_frame` and `.eh_frame`. These
sections contain DWARF CFI (unwinding) tables. I'm also not going to give a
complete description of these -- again the DWARF spec is great, read it --
but I will sketch out the basic idea in the next section.






## Unwinding Tables (DWARF CFI)

If you don't have nice function prologues with frame pointers -- or if you
want to know how to restore the other callee-saved registers, you need something
more. You need unwinding tables. There are various formats of these tables but
they all tell you how to do the same basic things:

* How to recover the return address for the current frame
* How to recover some of the registers for the current frame
* How to run destructors / catch the unwind at runtime (personality/LSDA, not covered in this article)

And indeed if we look at DWARF CFI, it's a giant table mapping every single address
in a module to those rules.

The rules to recover the return address/registers are basically a
complete virtual machine, because DWARF *really* wanted to let optimizers
and language designers go completely wild and have the most incredibly
convoluted unwinding methods possible.

Naively these unwinding tables would be *enormous* (far bigger than the executable
itself, as it would have multiple unwinding instructions for every instruction
in the executable), so a compression scheme is needed.

DWARF CFI achieves compression by allowing entries in the table to be a *diff*
on a base entry that covers a range of instructions. The simplest and most efficient diff
is just *not having a new entry*. Unless a new rule is introduced at an address,
the previous rule applies.

To simplify these diffs, DWARF CFI defines a "CFA" (canonical frame address)
register. This is conceptually the frame pointer, but doesn't necessarily have
to be. Since most rules end up being something to the effect of "`rbx` is saved
at offset 24 from the start of the frame", most of our rules end up looking like
`%rbx := *(%cfa + 24)`.

When we can do this, most of our entries will just be instructions on how to update the CFA.
Here's an example of what the CFI for a function which saves rbp and rbx would look like:

```text
initial rule (covering 0x08 to 0xA8):   (start of function)
    %rip := *(%cfa);
    %cfa := %rsp - 8

diff rule (starting at 0x10):           (function pushes rbp)
    %rbp := *(%cfa + 8)
    %cfa := %rsp - 16

diff rule (starting at 0x18):           (function pushes rbx)
    %rbx := *(%cfa + 16)
    %cfa := %rsp - 24

diff rule (starting at 0x20):           (function reserves remaining stack space)
    %cfa := %rsp - 120
```

Unfortunately this format isn't all roses and sunshine. Compilers can easily make mistakes
generating it, and it's apparently quite slow to evaluate. I quite liked the discussion
of these issues in the paper [Reliable and Fast DWARF-Based Stack Unwinding](https://fzn.fr/projects/frdwarf/frdwarf-oopsla19.pdf) *(Bastian, Kell, Nardelli -- Proc. ACM Program. Lang., Vol. 3, No. OOPSLA, Article 146. Publication date: October 2019)*.





## Stack Scanning

For completeness sake, here's the last strategy for unwinding. If you don't
have unwinding tables or frame pointers (or those seem to produce obviously-wrong results),
you can resort to *stack scanning*.

Stack scanning is incredibly simple: just start walking down the stack and reading
values out of it. If any of them look like a return address, assume that's the return
address (and the caller's stack pointer should be restored to that location).

It's a surprisingly decent fallback.

The main question to answer is "what does a return address"
look like. If you have other kinds of debug info or can lookup modules, then
you can check if the address maps to a module/function. Otherwise you can try
eliminating things like non-canonical addresses.



# The Compact Unwinding Format

Ok, now we can talk about Apple's Compact Unwinding Format, which is found in
a binary's `__unwind_info` section. As a reminder, this is based on [my
implementation of the format](https://github.com/getsentry/symbolic/blob/49bd892c2efbfcf502d20164dac942101ddb43cb/symbolic-debuginfo/src/macho/compact.rs).

I personally found llvm's terminology for compact unwinding very confusing,
so this document will use the following terminology changes:

* "encoding" or "encoding type" => opcode (how to unwind)
* "function offset" => instruction address (an address in the executable)

Like all unwinding info formats, the goal of the compact unwinding format
is to create a mapping from addresses in the binary to opcodes describing
how to unwind from that location.

These opcodes describe:

* How to recover the return pointer for the current frame
* How to recover some of the registers for the current frame
* How to run destructors / catch the unwind at runtime (personality/LSDA)

A user of the compact unwinding format would:

1. Get the current instruction pointer (e.g. `%rip`).
2. Lookup the corresponding opcode in the compact unwinding structure.
3. Follow the instructions of that opcode to recover the current frame.
4. Optionally perform runtime unwinding tasks for the current frame (destructors).
5. Use that information to recover the instruction pointer of the previous frame.
6. Repeat until unwinding is complete.

The compact unwinding format can be understood as two separate pieces:

* An architecture-agnostic "page-table" structure for finding opcode entries
* Architecture-specific opcode formats (x86, x64, and ARM64)

Unlike DWARF CFI, compact unwinding doesn't have facilities for incrementally
updating how to recover certain registers as the function progresses.

Empirical analysis suggests that there tends to only be one opcode for
an entire function (which explains why llvm refers to instruction addresses
as "function offsets"), although nothing in the format seems to *require*
this to be the case.

One consequence of only having one opcode for a whole function is that
functions will generally have incorrect instructions for the function's
prologue (where callee-saved registers are individually PUSHed onto the
stack before the rest of the stack space is allocated), and epilogue
(where callee-saved registers are individually POPed back into registers).

Presumably this isn't a very big deal, since there's very few situations
where unwinding would involve a function still executing its prologue/epilogue.
This might matter when handling a stack overflow that occurred while
saving the registers, or when processing a non-crashing thread in a minidump
that happened to be in its prologue/epilogue.

Similarly, the way ranges of instructions are mapped means that Compact
Unwinding will generally incorrectly map the padding bytes between functions
(attributing them to the previous function), while DWARF CFI tends to
more carefully exclude those addresses. Presumably also not a big deal.

Both of these things mean that if DWARF CFI and Compact Unwinding are
available for a function, the DWARF CFI is expected to be more precise.

It's possible that LSDA entries have addresses decoupled from the primary
opcode so that instructions on how to run destructors can vary more
granularly, but LSDA support is still TODO as it's not needed for
backtraces.


# Page Tables

This section describes the architecture-agnostic layout of the compact
unwinding format. The layout of the format is a two-level page-table
with one root first-level node pointing to arbitrarily many second-level
nodes, which in turn can hold several hundred opcode entries.

There are two high-level concepts in this format that enable significant
compression of the tables:

1. Eliding duplicate instruction addresses
2. Palettizing the opcodes



Trick 1 is standard for unwinders: the table of mappings is sorted by
address, and any entries that would have the same opcode as the
previous one are elided. So for instance the following:

```text
address: 1, opcode: 1
address: 2, opcode: 1
address: 3, opcode: 2
```

Is just encoded like this:

```text
address: 1, opcode: 1
address: 3, opcode: 2
```

We have found a few places with "zero-length" entries, where the same
address gets repeated, such as the following in `libsystem_kernel.dylib`:

```text
address: 0x000121c3, opcode: 0x00000000
address: 0x000121c3, opcode: 0x04000680
```

In this case you can just discard the zero-length one (the first one).



Trick 2 is more novel: At the first level a global palette of up to 127 opcodes
is defined. Each second-level "compressed" (leaf) page can also define up to 128 local
opcodes. Then the entries mapping instruction addresses to opcodes can use 8-bit
indices into those palettes instead of entire 32-bit opcodes. If an index is
smaller than the number of global opcodes, it's global, otherwise it's local
(subtract the global count to get the local index).

> Unclear detail: If the global palette is smaller than 127, can the local
palette be larger than 128?

To compress these entries into a single 32-bit value, the address is truncated
to 24 bits and packed with the index. The addresses stored in these entries
are also relative to a base address that each second-level page defines.
(This will be made more clear below).

There are also non-palletized "regular" second-level pages with absolute
32-bit addresses, but those are fairly rare. llvm seems to only want to emit
them in the final page.

The root page also stores the first address mapped by each second-level
page, allowing for more efficient binary search for a particular function
offset entry. (This is the base address the compressed pages use.)

The root page always has a final sentinel entry which has a null pointer
to its second-level page while still specifying a first address. This
makes it easy to lookup the maximum mapped address (the sentinel will store
that value +1), and just generally makes everything Work Nicer.



## Layout of the Page Table

The page table starts at the very beginning of the `__unwind_info` section
with the root page:

```rust,ignore
struct RootPage {
    /// Only version 1 is currently defined
    version: u32 = 1,

    /// The array of u32 global opcodes (offset relative to start of root page).
    ///
    /// These may be indexed by "compressed" second-level pages.
    global_opcodes_offset: u32,
    global_opcodes_len: u32,

    /// The array of u32 global personality codes
    /// (offset relative to start of root page).
    ///
    /// Personalities define the style of unwinding that an unwinder should
    /// use, and how to interpret the LSDA entries for a function (see below).
    personalities_offset: u32,
    personalities_len: u32,

    /// The array of FirstLevelPageEntry's describing the second-level pages
    /// (offset relative to start of root page).
    pages_offset: u32,
    pages_len: u32,

    // After this point there are several dynamically-sized arrays whose
    // precise order and positioning don't matter, because they are all
    // accessed using offsets like the ones above. The arrays are:

    global_opcodes: [u32; global_opcodes_len],
    personalities: [u32; personalities_len],
    pages: [FirstLevelPageEntry; pages_len],

    /// An array of LSDA pointers (Language Specific Data Areas).
    ///
    /// LSDAs are tables that an unwinder's personality function will use to
    /// find what destructors should be run and whether unwinding should
    /// be caught and normal execution resumed. We can treat them opaquely.
    ///
    /// Second-level pages have addresses into this array so that it can
    /// can be indexed, the root page doesn't need to know about them.
    lsdas: [LsdaEntry; unknown_len],
}


struct FirstLevelPageEntry {
    /// The first address mapped by this page.
    ///
    /// This is useful for binary-searching for the page that can map
    /// a specific address in the binary (the primary kind of lookup
    /// performed by an unwinder).
    first_address: u32,

    /// Offset to the second-level page (offset relative to start of root page).
    ///
    /// This may point to a RegularSecondLevelPage or a CompressedSecondLevelPage.
    /// Which it is can be determined by the 32-bit "kind" value that is at
    /// the start of both layouts.
    second_level_page_offset: u32,

    /// Base offset into the lsdas array that entries in this page will be
    /// relative to (offset relative to start of root page).
    lsda_index_offset: u32,
}


struct RegularSecondLevelPage {
    /// Always 2 (use to distinguish from CompressedSecondLevelPage).
    kind: u32 = 2,

    /// The Array of RegularEntry's (offset relative to **start of this page**).
    entries_offset: u16,
    entries_len: u16,
}


struct RegularEntry {
    /// The address in the binary for this entry (absolute).
    instruction_address: u32,
    /// The opcode for this address.
    opcode: u32,
}

struct CompressedSecondLevelPage {
    /// Always 3 (use to distinguish from RegularSecondLevelPage).
    kind: u32 = 3,

    /// The array of compressed u32 entries
    /// (offset relative to **start of this page**).
    ///
    /// Entries are a u32 that contains two packed values (from high to low):
    /// * 8 bits: opcode index
    ///   * 0..global_opcodes_len => index into global palette
    ///   * global_opcodes_len..255 => index into local palette
    ///     (subtract global_opcodes_len to get the real local index)
    /// * 24 bits: instruction address
    ///   * address is relative to this page's first_address!
    entries_offset: u16,
    entries_len: u16,

    /// The array of u32 local opcodes for this page
    /// (offset relative to **start of this page**).
    local_opcodes_offset: u16,
    local_opcodes_len: u16,
}


// TODO: why do these have instruction_addresses? Are they not in sync
// with the second-level entries?
struct LsdaEntry {
    instruction_address: u32,
    lsda_address: u32,
}
```



# Opcode Format

There are 3 architecture-specific opcode formats: x86, x64, and ARM64.

All 3 formats have a "null opcode" (`0x0000_0000`) which indicates that
there is no unwinding information for this range of addresses. This happens
with things like hand-written assembly subroutines. This implementation
will yield it as a valid opcode that converts into `CompactUnwindOp::None`.

All 3 formats share a common header in the top 8 bits (from high to low):

```rust,ignore
/// Whether this instruction is the start of a function.
is_start: u1,

/// Whether there is an lsda entry for this instruction.
has_lsda: u1,

/// An index into the global personalities array
/// (TODO: ignore if has_lsda == false?)
personality_index: u2,

/// The architecture-specific kind of opcode this is, specifying how to
/// interpret the remaining 24 bits of the opcode.
opcode_kind: u4,
```


## x86 and x64 Opcodes

x86 and x64 use the same opcode layout, differing only in the registers
being restored. Registers are numbered 0-6, with the following mappings:

x86:
* 0 => no register (like `Option::None`)
* 1 => `ebx`
* 2 => `ecx`
* 3 => `edx`
* 4 => `edi`
* 5 => `esi`
* 6 => `ebp`

x64:
* 0 => no register (like `Option::None`)
* 1 => `rbx`
* 2 => `r12`
* 3 => `r13`
* 4 => `r14`
* 5 => `r15`
* 6 => `rbp`

Note also that encoded sizes/offsets are generally divided by the pointer size
(since all values we are interested in are pointer-aligned), which of course differs
between x86 and x64.

There are 4 kinds of x86/x64 opcodes (specified by opcode_kind):

(One of the llvm headers refers to a 5th "0=old" opcode. Apparently this
was used for initial development of the format, and is basically just
reserved to prevent the testing data from ever getting mixed with real
data. Nothing should produce or handle it. It does incidentally match
the "null opcode", but it's fine to regard that as an unknown opcode
and do nothing.)


### x86/x64 Opcode Mode 1: Frame-Based

The function has the standard frame pointer (`bp`) prelude which:

* Pushes the caller's `bp` to the stack
* Sets `bp := sp` (new frame pointer is the current top of the stack)

`bp` has been preserved, and any callee-saved registers that need to be restored
are saved on the stack at a known offset from `bp`.

The return address is stored just before the caller's `bp`. The caller's stack
pointer should point before where the return address is saved.

Registers are stored in increasing order (so `reg1` comes before `reg2`).

If a register has the "no register" value, continue iterating the offset
forward. This lets the registers be stored slightly-non-contiguously on the
stack.

The remaining 24 bits of the opcode are interpreted as follows (from high to low):

```rust,ignore
/// Registers to restore (see register mapping above)
reg1: u3,
reg2: u3,
reg3: u3,
reg4: u3,
reg5: u3,
_unused: u1,

/// The offset from bp that the registers to restore are saved at,
/// divided by pointer size.
stack_offset: u8,
```



### x86/x64 Opcode Mode 2: Frameless (Stack-Immediate)

The callee's stack frame has a known size, so we can find the start
of the frame by offsetting from sp (the stack pointer). Any callee-saved
registers that need to be restored are saved at the start of the stack
frame.

The return address is saved immediately before the start of this frame. The
caller's stack pointer should point before where the return address is saved.

Registers are stored in *reverse* order on the stack from the order the
decoding algorithm outputs (so `reg[1]` comes before `reg[0]`).

If a register has the "no register" value, *do not* continue iterating the
offset forward -- registers are strictly contiguous (it's possible
"no register" can only be trailing due to the encoding, but I haven't
verified this).


The remaining 24 bits of the opcode are interpreted as follows (from high to low):

```rust,ignore
/// How big the stack frame is, divided by pointer size.
stack_size: u8,

_unused: u3,

/// The number of registers that are saved on the stack.
register_count: u3,

/// The permutation encoding of the registers that are saved
/// on the stack (see below).
register_permutations: u10,
```

The register permutation encoding is a Lehmer code sequence encoded into a
single variable-base number so we can encode the ordering of up to
six registers in a 10-bit space.

This can't really be described well with anything but code, so
just read this implementation or llvm's implementation for how to
encode/decode this.



### x86/x64 Opcode Mode 3: Frameless (Stack-Indirect)

(Currently Unimplemented)

Stack-Indirect is exactly the same situation as Stack-Immediate, but
the stack-frame size is too large for Stack-Immediate to encode. However,
the function prereserved the size of the frame in its prologue, so we can
extract the the size of the frame from a `sub` instruction at a known
offset from the start of the function (`subl $nnnnnnnn,ESP` in x86,
`subq $nnnnnnnn,RSP` in x64).

This requires being able to find the first instruction of the function
(TODO: presumably the first is_start entry <= this one?).

TODO: describe how to extract the value from the `sub` instruction.


```rust,ignore
/// Offset from the start of the function where the `sub` instruction
/// we need is stored. (NOTE: not divided by anything!)
instruction_offset: u8,

/// An offset to add to the loaded stack size, divided by pointer size.
/// This allows the stack size to differ slightly from the `sub`, to
/// compensate for any function prologue that pushes a bunch of
/// pointer-sized registers.
stack_adjust: u3,

/// The number of registers that are saved on the stack.
register_count: u3,

/// The permutation encoding of the registers that are saved on the stack
/// (see Stack-Immediate for a description of this format).
register_permutations: u10,
```

**Note**: apparently binaries generated by the clang in Xcode 6 generated
corrupted versions of this opcode, but this was fixed in Xcode 7
(released in September 2015), so *presumably* this isn't something we're
likely to encounter. But if you encounter messed up opcodes this might be why.



### x86/x64 Opcode Mode 4: Dwarf

There is no compact unwind info here, and you should instead use the
DWARF CFI in `.eh_frame` for this line. The remaining 24 bits of the opcode
are an offset into the `.eh_frame` section that should hold the DWARF FDE
for this instruction address.



## ARM64 Opcodes

ARM64 (AKA AArch64) is a lot more strict about the ABI of functions, and
as such it has fairly simple opcodes. There are 3 kinds of ARM64 opcode:



### ARM64 Opcode 2: Frameless

This is a "frameless" leaf function. The caller is responsible for
saving/restoring all of its general purpose registers. The frame pointer
is still the caller's frame pointer and doesn't need to be touched. The
return address is stored in the link register (`x30`). All we need to do is
pop the frame and move the return address back to the program counter (`pc`).

The remaining 24 bits of the opcode are interpreted as follows (from high to low):

```rust,ignore
/// How big the stack frame is, divided by 16.
stack_size: u12,

_unused: u12,
```



### ARM64 Opcode 3: Dwarf

There is no compact unwind info here, and you should instead use the
DWARF CFI in `.eh_frame` for this line. The remaining 24 bits of the opcode
are an offset into the `.eh_frame` section that should hold the DWARF FDE
for this instruction address.



### ARM64 Opcode 4: Frame-Based

This is a function with the standard prologue. The frame pointer (`x29`) and
return address (`pc`) were pushed onto the stack in a pair (ARM64 registers
are saved/restored in pairs), and then the frame pointer was updated
to the current stack pointer.

Any callee-saved registers that need to be restored were then pushed
onto the stack in pairs in the following order (if they were pushed at
all, see below):

1. `x19`, `x20`
2. `x21`, `x22`
3. `x23`, `x24`
4. `x25`, `x26`
5. `x27`, `x28`
6. `d8`, `d9`
7. `d10`, `d11`
8. `d12`, `d13`
9. `d14`, `d15`

The remaining 24 bits of the opcode are interpreted as follows (from high to low):

```rust,ignore
_unused: u15,

// Whether each register pair was pushed
d14_and_d15_saved: u1,
d12_and_d13_saved: u1,
d10_and_d11_saved: u1,
d8_and_d9_saved: u1,

x27_and_x28_saved: u1,
x25_and_x26_saved: u1,
x23_and_x24_saved: u1,
x21_and_x22_saved: u1,
x19_and_x20_saved: u1,
```



# Notable Corners

Here's some notable corner cases and esoterica of the format. Behaviour in
these situations is not strictly guaranteed (as in we may decide to
make the implemenation more strict or liberal if it is deemed necessary
or desirable). But current behaviour *is* documented here for the sake of
maintenance/debugging. Hopefully it also helps highlight all the ways things
can go wrong for anyone using this documentation to write their own tooling.

For all these cases, if an Error is reported during iteration/search, the
`CompactUnwindInfoIter` will be in an unspecified state for future queries.
It will never violate memory safety but it may start yielding chaotic
values.

If this implementation ever panics, that should be regarded as an
implementation bug.


Things we allow:

* The personalities array has a 32-bit length, but all indices into
it are only 2 bits. As such, it is theoretically possible for there
to be unindexable personalities. In practice that Shouldn't Happen,
and this implementation won't report an error if it does, because it
can be benign (although we have no way to tell if indices were truncated).

* The llvm headers say that at most there should be 127 global opcodes
and 128 local opcodes, but since local index translation is based on
the actual number of global opcodes and *not* 127/128, there's no
reason why each palette should be individually limited like this.
This implementation doesn't report an error if this happens, and should
work fine if it does.

* The llvm headers say that second-level pages are *actual* pages at
a fixed size of 4096 bytes. It's unclear what advantage this provides,
perhaps there's a situation where you're mapping in the pages on demand?
This puts a practical limit on the number of entries each second-level
page can hold -- regular pages can fit 511 entries, while compressed
pages can hold 1021 entries+local_opcodes (they have to share). This
implementation does not report an error if a second-level page has more
values than that, and should work fine if it does.

* If a `CompactUnwindInfoIter` is created for an architecture it wasn't
designed for, it is assumed that the layout of the page tables will
remain the same, and entry iteration/lookup should still work and
produce results. However `CompactUnwindInfoEntry::instructions`
will always return `CompactUnwindOp::None`.

* If an opcode kind is encountered that this implementation wasn't
designed for, `Opcode::instructions` will return `CompactUnwindOp::None`.

* If two entries have the same address (making the first have zero-length),
we silently discard the first one in favour of the second.

* Only 7 register mappings are provided for x86/x64 opcodes, but the
3-bit encoding allows for 8. This implementation will just map the
8th encoding to "no register" as well.

* Only 6 registers can be restored by the x86/x64 stackless modes, but
the 3-bit encoding of the register count allows for 7. This implementation
clamps the value to 6.


Things we produce errors for:

* If the root page has a version this implementation wasn't designed for,
`CompactUnwindInfoIter::new` will return an Error.

* A corrupt unwind_info section may have its entries out of order. Since
the next entry's instruction_address is always needed to compute the
number of bytes the current entry covers, the implementation will report
an error if it encounters this. However it does not attempt to fully
validate the ordering during an `entry_for_address` query, as this would
significantly slow down the binary search. In this situation
you may get chaotic results (same guarantees as `BTreeMap` with an
inconsistent `Ord` implementation).

* A corrupt unwind_info section may attempt to index out of bounds either
with out-of-bounds offset values (e.g. personalities_offset) or with out
of bounds indices (e.g. a local opcode index). When an array length is
provided, this implementation will return an error if an index is out
out of bounds. Offsets are only restricted to the unwind_info
section itself, as this implementation does not assume arrays are
placed in any particular place, and does not try to prevent aliasing.
Trying to access outside the `.unwind_info` section will return an error.

* If an unknown second-level page type is encountered, iteration/lookup will
return an error.


Things that cause chaos:

* If the null page was missing, there would be no way to identify the
number of instruction bytes the last entry in the table covers. As such,
this implementation assumes that it exists, and currently does not validate
it ahead of time. If the null page *is* missing, the last entry or page
may be treated as the null page, and won't be emitted. (Perhaps we should
provide more reliable behaviour here?)

* If there are multiple null pages, or if there is a page with a defined
second-level page but no entries of its own, behaviour is unspecified.

