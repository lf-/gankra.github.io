% Blogging about Rust using Rust's Compiler Toolchain

## Alexis Beingessner - May 23, 2015 - Rust 1.0.0

A while ago I had an annoying problem: I would like to write blog posts about Rust, but all I have at my disposal is a static file server that I can't modify on a system I have no priviledges on. I could post things on Github or like Mediumblr or something, but that didn't feel... right. I want to host my own words, damn it!

Now, I can definitely hand-write everything in html. At some limit, that's eveything I host. Still, I *really* like markdown. I'd also be writing a lot about code, which html is kinda janky at. I want to be lazy, damn it!

So like pandoc or something is probably a thing... I'd have to look that up... download stuff... install it... figure out how it works... nonono, all wrong! See the previous proclamation for details.

What I *did* know is that the Rust compiler has a nifty little too called rustdoc, for building documentation. It's most powerful feature is scraping an entire crate's source and building nice static html indexes. However more obscure is that you can just feed it markdown files, and it will spit out html. That's how TRPL is built. It comes with the rest of the compiler tools, so it's even already in my `$PATH`. Sounds like what I want!

The only special thing it requires from your file is a title line prefixed by a `%`. It works pretty ok on its own but the raw output is pretty... raw. No source highlighting. No CSS. On the other end of the spectrum, it produces this TOC that I really don't like. And spits the file out in a `doc/` folder (which it would just crash on the absence of!).

Thankfully, `rustdoc --help` is full of little goodies:

```text
> rustdoc --help
/Users/ABeingessner/.multirust/toolchains/nightly/bin/rustdoc [options] <input>

Options:
    -h --help           show this help message
    -V --version        print rustdoc's version
    -v --verbose        use verbose output
    -r --input-format [rust|json]
                        the input type of the specified file
    -w --output-format [html|json]
                        the output type to write
    -o --output PATH    where to place the output
    --crate-name NAME   specify the name of this crate
    -L --library-path DIR
                        directory to add to crate search path
    --cfg               pass a --cfg to rustc
    --extern NAME=PATH  pass an --extern to rustc
    --plugin-path DIR   directory to load plugins from
    --passes PASSES     list of passes to also run, you might want to pass it
                        multiple times; a value of `list` will print available
                        passes
    --plugins PLUGINS   space separated list of plugins to also load
    --no-defaults       don't run the default passes
    --test              run code examples as tests
    --test-args ARGS    arguments to pass to the test runner
    --target TRIPLE     target triple to document
    --markdown-css FILES
                        CSS files to include via <link> in a rendered Markdown
                        file
    --html-in-header FILES
                        files to include inline in the <head> section of a
                        rendered Markdown file or generated documentation
    --html-before-content FILES
                        files to include inline between <body> and the content
                        of a rendered Markdown file or generated documentation
    --html-after-content FILES
                        files to include inline between the content and
                        </body> of a rendered Markdown file or generated
                        documentation
    --markdown-playground-url URL
                        URL to send code snippets to
    --markdown-no-toc   don't include table of contents

```

Right at the bottom is a direct answer to one of my problems: `--markdown-no-toc`. For a bit I got by with the rest by just hand-editing the HTML. Paste in the CSS, add some extra html hooks for the CSS, etc. Unfortunately this was pretty brittle to editing the text!

Someone on the internet helpfully pointed out that my code *was* being marked up for code highlighting, I just didn't have the CSS for it. Having worked on rustdoc a bit myself, I knew roughly where that would have to be. A quick ctrl+f for `code` brought up [this](https://github.com/rust-lang/rust/blob/master/src/doc/rust.css#L224-L233):

```css
/* Code highlighting */
pre.rust .kw { color: #8959A8; }
pre.rust .kw-2, pre.rust .prelude-ty { color: #4271AE; }
pre.rust .number, pre.rust .string { color: #718C00; }
pre.rust .self, pre.rust .boolval, pre.rust .prelude-val,
pre.rust .attribute, pre.rust .attribute .ident { color: #C82829; }
pre.rust .comment { color: #8E908C; }
pre.rust .doccomment { color: #4D4D4C; }
pre.rust .macro, pre.rust .macro-nonterminal { color: #3E999F; }
pre.rust .lifetime { color: #B76514; }
```

which I promptly stole.

I still was hand-editing some stuff into the file, though. After a couple posts I decided the time waste had reach a critical mass and paid a little more attention to the above flags. The three `--html-*` flags give me exactly the places I was editing content into. Then I just tossed it all in a bash script so that I could forget how to do all of this *forever*:

```text
#!/bin/bash

mkdir doc
rustdoc --markdown-no-toc --html-in-header ../rustblog/head.html --html-before-content ../rustblog/before.html --html-after-content ../rustblog/after.html "$1.md"
mv -f "doc/$1.html" ./
rm -rf doc
```

And thus `rustblog` was born. Was it worth it? I don't even know. But heck, it works. [Here's this post's text](index.md).
