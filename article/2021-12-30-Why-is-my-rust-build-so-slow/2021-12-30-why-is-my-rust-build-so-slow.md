---
category: Rust
title: Why is my Rust build so slow?
date: "2020-12-30 00:00:00"
---

原文：https://fasterthanli.me/articles/why-is-my-rust-build-so-slow

I've recently come back to an older project of mine (that powers this website), and as I did some maintenance work: upgrade to newer crates, upgrade to [a newer rustc](https://fasterthanli.me/articles/why-is-my-rust-build-so-slow), I noticed that my build was taking too damn long!

For me, this is a big issue. Because I juggle a lot of things at any given time, and I have less and less time to just hyperfocus on an issue, I try to make my setup as productive as possible.


This is why, for work, I bought a [5950X](https://www.amd.com/en/products/cpu/amd-ryzen-9-5950X). It's no ThreadRipper, but it gives me 32 logical cores to play with, and most of the time, the computer is waiting on me, rather than the other way around.

【img】

But despite having a beefy CPU, an excessive amount of RAM (128GB), and a very fast SSD, my project took 2m09s for a "cold" release build, and 1m11s for a "hot" release build (changing a single line in main.rs and recompiling).

```shell
time cargo build --release

    Finished release [optimized + debuginfo] target(s) in 2m 09s
cargo build --release  890.49s user 117.77s system 779% cpu 2:09.31 total

echo "Editing some files..."
time cargo build --release

    Finished release [optimized + debuginfo] target(s) in 1m 11s
cargo build --release  127.99s user 8.42s system 191% cpu 1:11.32 total
```

It would make sense for the initial build to be somewhat long, as I'm pulling a lot of crates here, 334 to be exact:

```
cat Cargo.lock | toml2json | jq '.package | length'
334
```

> Cool bear's hot tip
> Cargo.lock is a TOML file that contains package entries for all used crates (even transitive dependencies). [toml2json](https://github.com/woodruffw/toml2json) is a Rust CLI utility that does exactly what it sounds like, and jq lets me count items in an array.

And it would make some sense for the "hot" release build to take some time, too because there's quite a bit of code in that main crate: I really need to split it up into several smaller crates to have clearer interfaces and make compilation faster (because crates can be compiled in parallel).

```
tokei -t=Rust

===============================================================================
 Language            Files        Lines         Code     Comments       Blanks
===============================================================================
 Rust                   54         7334         6483           30          821
 |- Markdown             8           31            0           31            0
 (Total)                           7365         6483           61          821
===============================================================================
 Total                  54         7334         6483           30          821
===============================================================================
```

> Cool bear's hot tip
> [tokei](https://lib.rs/crates/tokei) is a Rust CLI tool to count lines of code.

> Counting lines of code is a silly measure that should almost never be used, but let's keep it simple for now.

If you haven't been writing Rust for a long time, you've probably heard "Rust compile times are long" and so it might be hard for you to gauge whether "over one minute" is excessive for a hot build, luckily I'm here to tell you: *of course it's excessive*.

Anything over "a few seconds" is excessive.

So, how do we find what's going on?

## What is cargo even doing

In another language, say, C or C++, we might write a Makefile by hand.

> What? No. Nobody does that anymore.

Okay, well, you might write a file, that eventually generates a Makefile, or a set of instructions for ninja, or MSBuild, or xcodebuild, or whatever: the point is, at some point you would have a file to look at, that would tell you what steps are involved in building your thing.

And if you made a diagram out of it, it would look something like that:

【img】

And building Rust looks a lot like this!

It's just that, y'know, `cargo` is driving the whole process, so most of the time you don't really need to concern yourself with the details.

But if you want to concern yourself with the details, you can!

Say for example, we look inside the `target/` directory, we would find files like these:

```
find target -name '*.rlib' | head -5

target/release/deps/libmatches-db00cdc86371b34a.rlib
target/release/deps/libcfg_if-038689491f275bed.rlib
target/release/deps/libpin_project_lite-06e0655a601f73df.rlib
target/release/deps/libfnv-663a3fe7793aefd3.rlib
target/release/deps/libtinyvec_macros-4b4139e126989f5f.rlib
```
And these are just "ar archives" (the ar stands for "archive" already, it's a bit of an "ATM machine" situation), at least on Linux here:

```
file ./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib

./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib: current ar archive
```

And an ar archive just contains a bunch of `.o`:

```
ar t ./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib

tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.0.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.1.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.10.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.11.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.12.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.13.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.14.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.15.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.2.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.3.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.4.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.5.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.6.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.7.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.8.rcgu.o
tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.9.rcgu.o
lib.rmeta
```

And these, in turn, contain symbols: variables, functions, etc. and their associated code, already compiled, and ready to be linked:

```
nm ./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib | tail -8

nm: lib.rmeta: no symbols
0000000000000000 T _ZN4core3ptr68drop_in_place$LT$alloc..raw_vec..RawVec$LT$tree_sitter..Node$GT$$GT$17h79c5587a7f50bc54E.llvm.2637932016532741116
0000000000000000 t _ZN4core3ptr77drop_in_place$LT$alloc..vec..Vec$LT$tree_sitter_highlight..LocalScope$GT$$GT$17hb6e2dd6e6944713aE
                 U _ZN59_$LT$tree_sitter..Tree$u20$as$u20$core..ops..drop..Drop$GT$4drop17hee429551fa9c7144E
                 U _ZN66_$LT$tree_sitter..QueryCursor$u20$as$u20$core..ops..drop..Drop$GT$4drop17h94c27d3825e7214aE
0000000000000000 T _ZN86_$LT$alloc..vec..into_iter..IntoIter$LT$T$C$A$GT$$u20$as$u20$core..ops..drop..Drop$GT$4drop17h65a129b4029ecfd2E
0000000000000000 T _ZN86_$LT$alloc..vec..into_iter..IntoIter$LT$T$C$A$GT$$u20$as$u20$core..ops..drop..Drop$GT$4drop17he83a9f6dd95c6c07E
```

> Mhh that's not super readable.


True! We can pipe them into [rustfilt](https://crates.io/crates/rustfilt) to [demangle](https://en.wikipedia.org/wiki/Name_mangling) them.

```
nm ./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib | tail -8 | rustfilt
nm: lib.rmeta: no symbols
0000000000000000 T core::ptr::drop_in_place<alloc::raw_vec::RawVec<tree_sitter::Node>>
0000000000000000 t core::ptr::drop_in_place<alloc::vec::Vec<tree_sitter_highlight::LocalScope>>
                 U <tree_sitter::Tree as core::ops::drop::Drop>::drop
                 U <tree_sitter::QueryCursor as core::ops::drop::Drop>::drop
0000000000000000 T <alloc::vec::into_iter::IntoIter<T,A> as core::ops::drop::Drop>::drop
0000000000000000 T <alloc::vec::into_iter::IntoIter<T,A> as core::ops::drop::Drop>::drop

lib.rmeta:
```

Or just use nm's `--demangle` option:

```
nm --demangle ./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib | tail -8
nm: lib.rmeta: no symbols
0000000000000000 T _ZN4core3ptr68drop_in_place$LT$alloc..raw_vec..RawVec$LT$tree_sitter..Node$GT$$GT$17h79c5587a7f50bc54E.llvm.2637932016532741116
0000000000000000 t core::ptr::drop_in_place<alloc::vec::Vec<tree_sitter_highlight::LocalScope>>
                 U <tree_sitter::Tree as core::ops::drop::Drop>::drop
                 U <tree_sitter::QueryCursor as core::ops::drop::Drop>::drop
0000000000000000 T <alloc::vec::into_iter::IntoIter<T,A> as core::ops::drop::Drop>::drop
0000000000000000 T <alloc::vec::into_iter::IntoIter<T,A> as core::ops::drop::Drop>::drop

lib.rmeta:
```

LLVM also provides an `llvm-nm` utility, which works well (although it lists symbols in a different order, hence the `tail` => `head`):

```
llvm-nm --demangle ./target/release/deps/libtree_sitter_highlight-dbbf005203d40df6.rlib | head -8

tree_sitter_highlight-dbbf005203d40df6.tree_sitter_highlight.f125e660-cgu.0.rcgu.o:
0000000000000000 T core::ptr::drop_in_place$LT$core..alloc..layout..LayoutError$GT$::h01bcc4d6d4dc9891 (.llvm.1106350809534913491)
                 U core::panicking::panic::h0ba7146865b2f9d6
                 U alloc::alloc::handle_alloc_error::h6ad4108518320222
0000000000000000 T alloc::raw_vec::finish_grow::hca9a665eb9d6dbd6 (.llvm.1106350809534913491)
                 U alloc::raw_vec::capacity_overflow::h12238855ca9dc4ed
0000000000000000 T alloc::raw_vec::RawVec$LT$T$C$A$GT$::allocate_in::h19c1519794c5cb29
```

So we've got every component of our graph, except it looks more like this:

【img】

The notable differences here is that rustc is called once per crate. The object files are not "per source file", they're per "rust codegen unit" ([RCGU](https://github.com/rust-lang/rust/blob/3d57c61a9e04dcd3df633f41142009d6dcad4399/compiler/rustc_session/src/config.rs#L657)).

So:

- cargo invokes `rustc` once per crate
- `rustc` decides how many "codegen units" to do in parallel, writes them as `.o` files, then archives them in an `.rlib`
- cargo does one final `rustc` invocation with all the required `.rlib`, which ends up calling the linker to make a binary

> Cool bear's hot tip

> There's a bit of confusion around nomenclature here: something that has a Cargo.toml is a package. Packages are published on a package registry, like https://crates.io, and each package may have multiple crates: a build script crate, a lib crate, one or more bin crates, etc.

> Thanks to [@SimonSapin](https://twitter.com/SimonSapin) for clearing that up!

If we use cargo's `--verbose` flag, we can see it making separate rustc invocations: let's try it on a smaller crate so the output is a more manageable size.

```
cargo build --verbose
    Blocking waiting for file lock on build directory
       Fresh unicode-xid v0.2.2
       Fresh cc v1.0.72
   Compiling regex-syntax v0.6.25
       Fresh proc-macro2 v1.0.35
       Fresh quote v1.0.10
   Compiling memchr v2.4.1
     Running `rustc --crate-name regex_syntax --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/regex-syntax-0.6.25/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 --cfg 'feature="default"' --cfg 'feature="unicode"' --cfg 'feature="unicode-age"' --cfg 'feature="unicode-bool"' --cfg 'feature="unicode-case"' --cfg 'feature="unicode-gencat"' --cfg 'feature="unicode-perl"' --cfg 'feature="unicode-script"' --cfg 'feature="unicode-segment"' -C metadata=5f99d85e7aa20c3c -C extra-filename=-5f99d85e7aa20c3c --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --cap-lints allow -C link-arg=-fuse-ld=lld`
       Fresh syn v1.0.84
       Fresh thiserror-impl v1.0.30
     Running `rustc --crate-name memchr --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/memchr-2.4.1/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 --cfg 'feature="default"' --cfg 'feature="std"' -C metadata=6061dad8797913cc -C extra-filename=-6061dad8797913cc --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --cap-lints allow -C link-arg=-fuse-ld=lld --cfg memchr_runtime_simd --cfg memchr_runtime_sse2 --cfg memchr_runtime_sse42 --cfg memchr_runtime_avx`
   Compiling thiserror v1.0.30
     Running `rustc --crate-name thiserror --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/thiserror-1.0.30/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 -C metadata=e606936c310ae4d3 -C extra-filename=-e606936c310ae4d3 --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --extern thiserror_impl=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libthiserror_impl-b40be9909dd03ab5.so --cap-lints allow -C link-arg=-fuse-ld=lld`
   Compiling aho-corasick v0.7.18
     Running `rustc --crate-name aho_corasick --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/aho-corasick-0.7.18/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 --cfg 'feature="default"' --cfg 'feature="std"' -C metadata=d54b0d9e8dd84d99 -C extra-filename=-d54b0d9e8dd84d99 --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --extern memchr=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libmemchr-6061dad8797913cc.rmeta --cap-lints allow -C link-arg=-fuse-ld=lld`
   Compiling regex v1.5.4
     Running `rustc --crate-name regex --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/regex-1.5.4/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 --cfg 'feature="aho-corasick"' --cfg 'feature="default"' --cfg 'feature="memchr"' --cfg 'feature="perf"' --cfg 'feature="perf-cache"' --cfg 'feature="perf-dfa"' --cfg 'feature="perf-inline"' --cfg 'feature="perf-literal"' --cfg 'feature="std"' --cfg 'feature="unicode"' --cfg 'feature="unicode-age"' --cfg 'feature="unicode-bool"' --cfg 'feature="unicode-case"' --cfg 'feature="unicode-gencat"' --cfg 'feature="unicode-perl"' --cfg 'feature="unicode-script"' --cfg 'feature="unicode-segment"' -C metadata=84dc590185c6397b -C extra-filename=-84dc590185c6397b --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --extern aho_corasick=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libaho_corasick-d54b0d9e8dd84d99.rmeta --extern memchr=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libmemchr-6061dad8797913cc.rmeta --extern regex_syntax=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libregex_syntax-5f99d85e7aa20c3c.rmeta --cap-lints allow -C link-arg=-fuse-ld=lld`
   Compiling tree-sitter v0.20.1
     Running `rustc --crate-name tree_sitter --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tree-sitter-0.20.1/binding_rust/lib.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 -C metadata=434249d5421db9dc -C extra-filename=-434249d5421db9dc --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --extern regex=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libregex-84dc590185c6397b.rmeta --cap-lints allow -C link-arg=-fuse-ld=lld -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-97060a66e399d154/out -l static=tree-sitter`
   Compiling tree-sitter-highlight v0.20.1
     Running `rustc --crate-name tree_sitter_highlight --edition=2018 /home/amos/.cargo/registry/src/github.com-1ecc6299db9ec823/tree-sitter-highlight-0.20.1/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --crate-type staticlib --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=f889da5f4ac6bce2 -C extra-filename=-f889da5f4ac6bce2 --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --extern regex=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libregex-84dc590185c6397b.rlib --extern thiserror=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libthiserror-e606936c310ae4d3.rlib --extern tree_sitter=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libtree_sitter-434249d5421db9dc.rlib --cap-lints allow -C link-arg=-fuse-ld=lld -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-97060a66e399d154/out`
   Compiling tree-sitter-collection v0.26.0 (/home/amos/bearcove/tree-sitter-collection)
     Running `rustc --crate-name tree_sitter_collection --edition=2018 src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 -C metadata=a567e6662873a172 -C extra-filename=-a567e6662873a172 --out-dir /home/amos/bearcove/tree-sitter-collection/target/debug/deps -C incremental=/home/amos/bearcove/tree-sitter-collection/target/debug/incremental -L dependency=/home/amos/bearcove/tree-sitter-collection/target/debug/deps --extern tree_sitter=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libtree_sitter-434249d5421db9dc.rmeta --extern tree_sitter_highlight=/home/amos/bearcove/tree-sitter-collection/target/debug/deps/libtree_sitter_highlight-f889da5f4ac6bce2.rlib -C link-arg=-fuse-ld=lld -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-collection-c4ebc5be2f6a2943/out -l static=tree-sitter-go -l static=tree-sitter-c -l static=tree-sitter-rust -l static=tree-sitter-javascript -l static=tree-sitter-java -l static=tree-sitter-typescript -l static=tree-sitter-tsx -l static=tree-sitter-toml -l static=tree-sitter-bash-parser -l static=tree-sitter-bash-scanner -l stdc++ -l static=tree-sitter-html-parser -l static=tree-sitter-html-scanner -l stdc++ -l static=tree-sitter-python-parser -l static=tree-sitter-python-scanner -l stdc++ -l static=tree-sitter-ini-parser -l static=tree-sitter-meson-parser -l static=tree-sitter-yaml-parser -l static=tree-sitter-yaml-scanner -l stdc++ -l static=tree-sitter-dockerfile-parser -L native=/home/amos/bearcove/tree-sitter-collection/target/debug/build/tree-sitter-97060a66e399d154/out`
    Finished dev [unoptimized + debuginfo] target(s) in 9.95s
```

Yay!

And if we don't trust `cargo`, instead of asking nicely, we can also just straight up spy on it, with `strace`.

Let's try it on an even smaller crate, because that one's pretty noisy:



> Cool bear's hot tip

> In the invocation below, we trace system calls (strace), including children processes (following forks, `-f`), for the execution of `cargo build --quiet`. We then redirect standard error (stderr, file descriptor 2) to standard output (stdout, file descriptor 1), and pipe into [grep](https://en.wikipedia.org/wiki/Grep), using "extended" regular expression syntax (-E), and we look for something that starts with `execve` (and ends with = 0 (which indicates success).

```
$ cargo clean && strace -f -e execve -- cargo build --quiet 2>&1 | grep -E 'execve\(.*= 0'
execve("/home/amos/.cargo/bin/cargo", ["cargo", "build", "--quiet"], 0x7fff74a9bec0 /* 51 vars */) = 0
execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/cargo", ["/home/amos/.rustup/toolchains/st"..., "build", "--quiet"], 0x5628508c5b60 /* 56 vars */) = 0
[pid 702423] execve("/home/amos/.cargo/bin/rustc", ["rustc", "-vV"], 0x5604183fdd60 /* 58 vars */) = 0
[pid 702423] execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc", ["/home/amos/.rustup/toolchains/st"..., "-vV"], 0x55d9605a1250 /* 58 vars */) = 0
[pid 702424] execve("/home/amos/.cargo/bin/rustc", ["rustc", "-", "--crate-name", "___", "--print=file-names", "--crate-type", "bin", "--crate-type", "rlib", "--crate-type", "dylib", "--crate-type", "cdylib", "--crate-type", "staticlib", "--crate-type", "proc-macro", "-Csplit-debuginfo=packed"], 0x5604184cac20 /* 58 vars */) = 0
[pid 702424] execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc", ["/home/amos/.rustup/toolchains/st"..., "-", "--crate-name", "___", "--print=file-names", "--crate-type", "bin", "--crate-type", "rlib", "--crate-type", "dylib", "--crate-type", "cdylib", "--crate-type", "staticlib", "--crate-type", "proc-macro", "-Csplit-debuginfo=packed"], 0x55eaa32ffb50 /* 58 vars */) = 0
[pid 702425] execve("/home/amos/.cargo/bin/rustc", ["rustc", "-", "--crate-name", "___", "--print=file-names", "--crate-type", "bin", "--crate-type", "rlib", "--crate-type", "dylib", "--crate-type", "cdylib", "--crate-type", "staticlib", "--crate-type", "proc-macro", "--force-warn=rust-2021-compatibi"...], 0x56041841ba80 /* 58 vars */) = 0
[pid 702425] execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc", ["/home/amos/.rustup/toolchains/st"..., "-", "--crate-name", "___", "--print=file-names", "--crate-type", "bin", "--crate-type", "rlib", "--crate-type", "dylib", "--crate-type", "cdylib", "--crate-type", "staticlib", "--crate-type", "proc-macro", "--force-warn=rust-2021-compatibi"...], 0x559d2d9dc7b0 /* 58 vars */) = 0
[pid 702427] execve("/home/amos/.cargo/bin/rustc", ["rustc", "-", "--crate-name", "___", "--print=file-names", "--crate-type", "bin", "--crate-type", "rlib", "--crate-type", "dylib", "--crate-type", "cdylib", "--crate-type", "staticlib", "--crate-type", "proc-macro", "--print=sysroot", "--print=cfg"], 0x5604184cac20 /* 58 vars */) = 0
[pid 702427] execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc", ["/home/amos/.rustup/toolchains/st"..., "-", "--crate-name", "___", "--print=file-names", "--crate-type", "bin", "--crate-type", "rlib", "--crate-type", "dylib", "--crate-type", "cdylib", "--crate-type", "staticlib", "--crate-type", "proc-macro", "--print=sysroot", "--print=cfg"], 0x55e0804f1d00 /* 58 vars */) = 0
[pid 702429] execve("/home/amos/.cargo/bin/rustc", ["rustc", "-vV"], 0x5604183fdd60 /* 58 vars */) = 0
[pid 702429] execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc", ["/home/amos/.rustup/toolchains/st"..., "-vV"], 0x55ffc26bb690 /* 58 vars */) = 0
[pid 702432] execve("/home/amos/.cargo/bin/rustc", ["rustc", "--crate-name", "hello_world", "--edition=2021", "src/main.rs", "--error-format=json", "--json=diagnostic-rendered-ansi", "--crate-type", "bin", "--emit=dep-info,link", "-C", "embed-bitcode=no", "-C", "debuginfo=2", "-C", "metadata=edf9f7e82579517e", "-C", "extra-filename=-edf9f7e82579517e", "--out-dir", "/home/amos/bearcove/hello-world/"..., "-C", "incremental=/home/amos/bearcove/"..., "-L", "dependency=/home/amos/bearcove/h"...], 0x7f39d8004c30 /* 76 vars */) = 0
[pid 702432] execve("/home/amos/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc", ["/home/amos/.rustup/toolchains/st"..., "--crate-name", "hello_world", "--edition=2021", "src/main.rs", "--error-format=json", "--json=diagnostic-rendered-ansi", "--crate-type", "bin", "--emit=dep-info,link", "-C", "embed-bitcode=no", "-C", "debuginfo=2", "-C", "metadata=edf9f7e82579517e", "-C", "extra-filename=-edf9f7e82579517e", "--out-dir", "/home/amos/bearcove/hello-world/"..., "-C", "incremental=/home/amos/bearcove/"..., "-L", "dependency=/home/amos/bearcove/h"...], 0x5578142c7ba0 /* 76 vars */) = 0
[pid 702446] execve("/usr/bin/cc", ["cc", "-m64", "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "-Wl,--as-needed", "-L", "/home/amos/bearcove/hello-world/"..., "-L", "/home/amos/.rustup/toolchains/st"..., "-Wl,--start-group", "-Wl,-Bstatic", "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., "/home/amos/.rustup/toolchains/st"..., ...], 0x7f712c3bcd00 /* 78 vars */) = 0
[pid 702447] execve("/usr/libexec/gcc/x86_64-redhat-linux/11/collect2", ["/usr/libexec/gcc/x86_64-redhat-l"..., "-plugin", "/usr/libexec/gcc/x86_64-redhat-l"..., "-plugin-opt=/usr/libexec/gcc/x86"..., "-plugin-opt=-fresolution=/tmp/cc"..., "--build-id", "--no-add-needed", "--eh-frame-hdr", "--hash-style=gnu", "-m", "elf_x86_64", "-dynamic-linker", "/lib64/ld-linux-x86-64.so.2", "-pie", "-o", "/home/amos/bearcove/hello-world/"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "-L/home/amos/bearcove/hello-worl"..., "-L/home/amos/.rustup/toolchains/"..., "-L/home/amos/.rustup/toolchains/"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/lib/../lib64", "-L/usr/lib/../lib64", "-L/usr/lib/gcc/x86_64-redhat-lin"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., ...], 0x1ece9c0 /* 85 vars */) = 0
[pid 702448] execve("/usr/bin/ld", ["/usr/bin/ld", "-plugin", "/usr/libexec/gcc/x86_64-redhat-l"..., "-plugin-opt=/usr/libexec/gcc/x86"..., "-plugin-opt=-fresolution=/tmp/cc"..., "--build-id", "--no-add-needed", "--eh-frame-hdr", "--hash-style=gnu", "-m", "elf_x86_64", "-dynamic-linker", "/lib64/ld-linux-x86-64.so.2", "-pie", "-o", "/home/amos/bearcove/hello-world/"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "-L/home/amos/bearcove/hello-worl"..., "-L/home/amos/.rustup/toolchains/"..., "-L/home/amos/.rustup/toolchains/"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/lib/../lib64", "-L/usr/lib/../lib64", "-L/usr/lib/gcc/x86_64-redhat-lin"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., ...], 0x7ffc58148430 /* 85 vars */) = 0
```

Yay! We see everything. It does indeed calls `rustc` a bunch of times, and the final invocation calls `cc`, which calls [collect2](https://gcc.gnu.org/onlinedocs/gccint/Collect2.html) and eventually ld.

And if we ask it to use `lld`, LLVM's linker, instead, it does that!


```
cargo clean && RUSTFLAGS="-C link-args=-fuse-ld=lld" strace -f -e execve -- cargo build --quiet 2>&1 | grep -E 'execve\(.*= 0'

(cut)
[pid 705269] execve("/usr/libexec/gcc/x86_64-redhat-linux/11/collect2", ["/usr/libexec/gcc/x86_64-redhat-l"..., "-plugin", "/usr/libexec/gcc/x86_64-redhat-l"..., "-plugin-opt=/usr/libexec/gcc/x86"..., "-plugin-opt=-fresolution=/tmp/cc"..., "--build-id", "--no-add-needed", "--eh-frame-hdr", "--hash-style=gnu", "-m", "elf_x86_64", "-dynamic-linker", "/lib64/ld-linux-x86-64.so.2", "-pie", "-fuse-ld=lld", "-o", "/home/amos/bearcove/hello-world/"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "-L/home/amos/bearcove/hello-worl"..., "-L/home/amos/.rustup/toolchains/"..., "-L/home/amos/.rustup/toolchains/"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/lib/../lib64", "-L/usr/lib/../lib64", "-L/usr/lib/gcc/x86_64-redhat-lin"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., ...], 0x6e2970 /* 86 vars */) = 0
                                👇
[pid 705270] execve("/usr/bin/ld.lld", ["/usr/bin/ld.lld", "-plugin", "/usr/libexec/gcc/x86_64-redhat-l"..., "-plugin-opt=/usr/libexec/gcc/x86"..., "-plugin-opt=-fresolution=/tmp/cc"..., "--build-id", "--no-add-needed", "--eh-frame-hdr", "--hash-style=gnu", "-m", "elf_x86_64", "-dynamic-linker", "/lib64/ld-linux-x86-64.so.2", "-pie", "-o", "/home/amos/bearcove/hello-world/"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "/usr/lib/gcc/x86_64-redhat-linux"..., "-L/home/amos/bearcove/hello-worl"..., "-L/home/amos/.rustup/toolchains/"..., "-L/home/amos/.rustup/toolchains/"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/usr/lib/gcc/x86_64-redhat-lin"..., "-L/lib/../lib64", "-L/usr/lib/../lib64", "-L/usr/lib/gcc/x86_64-redhat-lin"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., "/home/amos/bearcove/hello-world/"..., ...], 0x7ffd9119d598 /* 86 vars */) = 0
```

## How much time are we spending on these steps?

That's all really nice, but... it doesn't really explain why our build was slow.

For starters, unless we run the build with `--verbose` and watch the terminal to monitor how long different parts are taking, we don't really know anything about timings.

And that's why cargo has a `-Z` timings option!

It's nightly-only, so you really should not use the trick I'm about to use and just use nightly instead, but since I really want to use it in stable instead, I'm going to export ``=1` and have stable pretend it's a nightly build.

> Cool bear's hot tip
> This works because stable builds also have all the nightly code compiled in: it's just feature-gated. No, not like [LaunchDarkly feature flags](https://www.youtube.com/watch?v=-4w52waLTx4), just, it'll detect you're trying to use a feature, parse it correctly and everything, and gently let you know that feature isn't stable yet.

> But rustc uses unstable features internally, so when bootstrapping, it needs to be able to compile itself, unstable features and all. That's why the `RUSTC_BOOTSTRAP` environment variable exists.

> Of course here we're horribly misusing it, and it's going to make a bunch of people sad, but I hope with time they can forgive us.

```
cargo clean && RUSTC_BOOTSTRAP=1 cargo build --release --quiet -Z timings
```

That took forever, but now we have... a `cargo-timing.html` file!

It's interactive and stuff, so I can't really do it justice, but I still want to show you what it looks like, so here goes, non-dark-mode and non-hidpi-friendly screenshots and all.

First we have a nice summary:

【img】

"Fresh" units would be those that we don't need to recompile, but this is a "cold" build, since we ran `cargo clean` beforehand.

Then we have a timeline! That shows aaaaaall 420 compilation units:

【img】

对话：

- Okay be real with me: did you add unnecessary dependencies just so the total would be 420?

- I swear I didn't.

- Nice.

420 rows don't really fit on my screen, and if they did, they'd be too small to be readable, but luckily, we can crank up that "Min unit time" slider to hide the really quick ones:

【img】

> Cool bear's hot tip

> What do the colors mean? I'm so glad you asked!

> **Orange** means we're running a build script. Everything else has to deal with pipelining: cargo always tries to build as many crates as possible in parallel, but it can only start the "next" crate if there's enough metadata available about all its dependencies.

> So, the **light blue** parts are where we're still busy generating metadata, and any dependents (crates that depend on us) are blocked. When transitioning to **lavender**, dependents are unblocked and can start building immediately.

> You can see this happening here: `regex-syntax` transitions from light blue to lavender and `regex` (line 16) immediately starts building: the dotted line shows the dependency:

【img】

You can read [more about pipelining](https://github.com/rust-lang/compiler-team/blob/master/content/working-groups/pipelining/NOTES.md) on the Rust compiler team's repository.

And here we can see, at a glance, the most expensive crates: `tree-sitter-collection`, one of my own crates, which compiles, like, 12 different tree-sitter parsers (as C code, with gcc), `libsqlite3-sys`, which builds all of sqlite3 (so I don't have to have it installed on whichever system I run this binary), `libquickjs-sys`, which is a whole-ass JavaScript engine (which I use for [KaTeX](https://en.wikipedia.org/wiki/KaTeX)).

And then the last hugeeeeeee light blue bar, that's just the binary crate we're compiling. (And that is the crux of my issue).

For completeness, here's what else it shows us: a CPU usage and concurrency graph:

【img】

You can see that CPU usage is really high for the first third of the compilation, and it spikes again at the end, but there's a big valley in between, where it appeares to be doing very little.

"Very little" is misleading, of course, since there's 32 logical cores, we're actually looking at one core being at full utilization. Still, the system as a whole is "mostly idle".

Finally, it gives us a table, sort of a hall of fame of slow-to-build crates:

【img】

Now for a hot build! If we change a single line in `main.rs` and rebuild, we get:

【img】

Which is almost completely devoid of insight.

Because it's a bin crate, `cargo -Z timings` doesn't report codegen time, and (at least up until the time of this writing), it doesn't report link time either, so... we have a big opaque block of 72 seconds, doing... something.

> What did we learn?
> If you want to visualize "how parallel" your build is, or "where cargo is spending time", you can use `-Z timings`. It's a nightly-only option, but if you're not afraid of code crime jail, you can (ab)use RUSTC_BOOTSTRAP to use it on stable instead.

> **DO NOT USE RUSTC_BOOTSTRAP TO MAKE PRODUCTION BUILDS**. Consider the output of any `RUSTC_BOOTSTRAP=1` build radioactive. It's only really forgivable if 1) you're bootstrapping the Rust compiler, or 2) you're just measuring something, like cargo timings, or unused dependencies with cargo-udeps.

## Linker, is it you?

For very large Rust projects, and for hot builds, it's not uncommon for linking to make up most of the "build time". And that's why people tend to reach for `lld` (LLVM's linker), for example.

Thing is... we're already using lld.

Here's the graph with GNU ld instead (the default for this system):

【img】

That is the difference `lld` makes here: it's three seconds faster than GNU ld for this build (out of some unknown "link duration" — we could measure it, but I can't be bothered right now).

Using `mold` instead, we have this:

【img】

Barely faster than `lld`. So linking is not the bottleneck here.

> What did we learn?

> When building larger applications, linking can become the slowest / most expensive part of a rebuild. GNU ld hasn't been the best option for a while now.

> Rui Ueyama changed the game, twice, by working on lld, LLVM's linker, and then on mold, which will be [recognized officially by GCC](https://twitter.com/rui314/status/1476168056766623745) soon!


## Debug symbols, perhaps?

By default, no debug symbols are included in "release" cargo builds.

Debug symbols allow mapping memory addresses to "source locations", among other things. Which in turns, unlocks comfortable ways of debugging our executable.

For example, with debug information, we can ask the debugger to stop execution on a specific function by adding a breakpoint on a function name:

```
rust-gdb --quiet --args futile serve
Reading symbols from futile...
(gdb) set print thread-events off
(gdb) break futile::Config::read
Breakpoint 1 at 0x7d105a: file src/main.rs, line 69.
(gdb) r
Starting program: /home/amos/.cargo/bin/futile serve
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Thread 1 "futile" hit Breakpoint 1, futile::Config::read () at src/main.rs:69
69              let config_path = if config_path.is_dir() {
(gdb)
```

Well, let's try it!

```
rust-gdb --quiet --args futile serve

Reading symbols from futile...
(No debugging symbols found in futile)
(gdb) set print thread-events off
(gdb) break futile::Config::read
Function "futile::Config::read" not defined.
Make breakpoint pending on future shared library load? (y or [n]) y
Breakpoint 1 (futile::Config::read) pending.
(gdb) r
Starting program: /home/amos/.cargo/bin/futile serve
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
 INFO  futile > Reading config from ./futile.toml
 INFO  futile > No config file found, will use defaults
 INFO  futile > Base dir is /home/amos/bearcove/fasterthanli.me
 WARN  futile > sqlite: [283] recovered 2 frames from WAL file /home/amos/bearcove/fasterthanli.me/.futile/content.db-wal
(etc.)
```

No!

> But I thought... don't ELF files (like executable and dynamic libraries, even static libraries) have symbol tables, that are distinct from debug info?

They do! If we look at `libm.so` (lib math), it has a bunch of symbols. Over a thousand, in fact:

```
nm --dynamic --defined-only /usr/lib64/libm.so.6 | head -10
00000000000119d0 W acos@@GLIBC_2.2.5
0000000000015370 W acosf@@GLIBC_2.2.5
0000000000064bc0 W acosf128@@GLIBC_2.26
0000000000046510 T __acosf128_finite@GLIBC_2.26
0000000000015370 W acosf32@@GLIBC_2.27
00000000000119d0 W acosf32x@@GLIBC_2.27
00000000000119d0 W acosf64@@GLIBC_2.27
0000000000010170 W acosf64x@@GLIBC_2.27
0000000000038310 T __acosf_finite@GLIBC_2.15
0000000000025630 i __acos_finite@GLIBC_2.15

nm --dynamic --defined-only /usr/lib64/libm.so.6 | wc -l
1102
```

Our executable however, doesn't define any symbols:

```
nm --dynamic --defined-only $(which futile)
```

It expects symbols to be defined by other dynamic libraries it depends on, though!

```
nm --dynamic --undefined-only $(which futile) | head
              U abort@GLIBC_2.2.5
              U accept4@GLIBC_2.10
              U access@GLIBC_2.2.5
              U acos@GLIBC_2.2.5
              U acosh@GLIBC_2.2.5
              U asin@GLIBC_2.2.5
              U asinh@GLIBC_2.2.5
              U __assert_fail@GLIBC_2.2.5
              U atan@GLIBC_2.2.5
              U atan2@GLIBC_2.2.5
```


> Cool bear's hot tip

> Note the difference: we were using `--defined-only`, now we're using `--undefined-only`, emphasis on the privative un prefix.

Look, math functions! I guess it links against `libm.so`...

```
objdump -p $(which futile) | grep 'NEEDED'
  NEEDED               libstdc++.so.6
  NEEDED               libgcc_s.so.1
  NEEDED               libm.so.6
  NEEDED               libc.so.6
  NEEDED               ld-linux-x86-64.so.2
```

Yes it does.

So symbols are only exported if they need to be exported. In the case of a library, we need to be able to call `acos` and `asin` from somewhere else, and so they're in the table. But for an executable, all we need to know (sort of), is where the code starts, and that's the "start address" here:

```
objdump -f $(which futile)

/home/amos/.cargo/bin/futile:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000150:
HAS_SYMS, DYNAMIC, D_PAGED
start address 0x0000000000635ec0
```

> Okay, so futile::Config::read doesn't need to be "exported", because it's only ever called "internally" from futile into futile, and... so how did GDB ever successfully set a breakpoint there?

It looked at debug information instead!

Because debug information is so useful, and because I like to be able to look at stack traces with "source location" (file name and line number) information, and because I like to be able to step through code in a debugger, even on release builds, I have this:


```
# in `futile/Cargo.toml`

[profile.release]
debug = 1
```


And so my executable has a bunch of additional ELF sections:


```
objdump --wide --headers $(which futile) | grep -F '.debug'
 12 .debug_gdb_scripts    00000022  00000000004a8b99  00000000004a8b99  004a8b99  2**0  CONTENTS, ALLOC, LOAD, READONLY, DATA
 32 .debug_abbrev         0006a4d3  0000000000000000  0000000000000000  0164f4dc  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 33 .debug_info           0195631c  0000000000000000  0000000000000000  016b99af  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 34 .debug_aranges        00108190  0000000000000000  0000000000000000  0300fcd0  2**4  CONTENTS, READONLY, DEBUGGING, OCTETS
 35 .debug_ranges         00d770e0  0000000000000000  0000000000000000  03117e60  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 36 .debug_str            016b3e43  0000000000000000  0000000000000000  03e8ef40  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 37 .debug_pubnames       0154980f  0000000000000000  0000000000000000  05542d83  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 38 .debug_pubtypes       0000690c  0000000000000000  0000000000000000  06a8c592  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 39 .debug_line           00da927e  0000000000000000  0000000000000000  06a92e9e  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 40 .debug_loclists       00453d2b  0000000000000000  0000000000000000  0783c11c  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 41 .debug_rnglists       0009618f  0000000000000000  0000000000000000  07c8fe47  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 42 .debug_line_str       000025c6  0000000000000000  0000000000000000  07d25fd6  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 43 .debug_macro          00002a41  0000000000000000  0000000000000000  07d2859c  2**0  CONTENTS, READONLY, DEBUGGING, OCTETS
 44 .debug_frame          000001d0  0000000000000000  0000000000000000  07d2afe0  2**3  CONTENTS, READONLY, DEBUGGING, OCTETS
```

And, the format of those is quite complicated, but with a little help from our friends we can trace back what GDB used to set a breakpoint on `futile::Config::read`:

```
$ readelf --debug-dump=info $(which futile) | rustfilt | grep 'futile::Config::read'
    <2e68b4>   DW_AT_linkage_name: (indirect string, offset: 0xb960a1): futile::Config::read
```

> Cool bear's hot tip


> Note that rustfilt is necessary here, otherwise we have the mangled form `_ZN6futile6Config4read17h2622e8ecc143c293E` instead.

> Alternatively, we could've piped into `grep -E 'futile.Config.read'`, which would have searched for the words `futile`, `Config`, and `read` separated by any single character.

But here's the point I'm getting at: generating that debug information is not free. It takes time!

In fact, the `profile.<name>.debug` cargo configuration key is not a simple on/off switch: there's levels 0 (or false), 1 and 2 (or true).

Most of the time, 2 really is overkill. Let's compare timings just so you can be convinced that it really does make an impact on the build times of `futile`:

- With `debug = 0`, a cold build takes ~1m55s
- With `debug = 1`, a cold build takes ~2m04s
- With `debug = 2`, a cold build takes ~2m12s

Now keep in mind we have a block of roughly ~72s where we're not sure what the compiler is doing, and the rest is embarassingly parallel, so the difference there is less impressive than it should be. We'll try again once we fix whatever is taking so long.




> What did we learn?

> Debug information is super duper useful, if only so when you get a crash, you can easily map a stack trace (a bunch of memory addresses) to a bunch of source locations (file name / line number, but also function name, etc).

> In `Cargo.toml`, `debug = true` actually means `debug = 2`, and it's usually overkill, unless you're doing the sort of debugging where you need to be able to inspect the value of local variables for example. If all you're after is a stack trace, `debug = 1` is good enough.

> If you think debug information is too large, first off, you're correct, but secondly, try reaching for objcopy `--compress-debug-sections` to compress it instead of throwing it away with `strip`: you might find that you're happy with that compromise.

## Incremental builds


One thing I've made sure to mention since the beginning of this article is that I'm talking about release builds, not debug builds. That's why all cargo invocations have had the `--release` flag.

Is it as slow to make a debug build of `futile`?

- Cold debug build: 1m33s (vs 2m04s for release)
- Hot debug build: 19s (vs 1m11s for release)

Huh indeed! So again, we expect there to be a difference: in debug builds, we do very few optimizations, whereas in release builds, we try to optimize as much as possible.

And optimizing isn't free, so, it makes sense that a release build would take longer. But that much longer?

There's another difference: by default, cargo's debug profile enables [incremental builds](https://doc.rust-lang.org/cargo/reference/profiles.html#incremental). And cargo's release profile has incremental builds disabled.

What happens if we enable incremental builds for release?

```
# in `futile/Cargo.toml`

[profile.release]
debug = 1
incremental = true
```

- Cold incremental build: 2m08s (vs 2m04s non-incremental)
- Hot incremental build: 20s (vs 1m11s non-incremental)

Yes it is! That's the whole trade-off of incremental builds: cold builds are a little slower, but hot builds should be faster.

> Mhh I'm looking at the docs here and... it says something about "codegen units" too, and it being different for incremental builds vs non-incremental builds?

Yes! The goal of "incremental builds" is to be able to recompile fast. It helps if compiling is fast in general, and if we only have to recompile as little code as possible.

So, with incremental builds, crates are split into more, smaller codegen units. In 1.57.0 at least, non-incremental builds default to only 16 codegen units, whereas incremental builds default to 256 codegen units.

> Could it be what's making the difference? The number of codegen units, and not incremental builds?

Let's try it!

```
# in `futile/Cargo.toml`

[profile.release]
debug = 1
incremental = false
codegen-units = 256
```

- Cold 256-cgu build: 2m02s (vs 2m04s for 16-cgu)
- Hot 256-cgu build: 1m11s (vs 1m11s for 16-cgu)

Nope. No significant difference.

> Incremental builds sound great. Why wouldn't we want to use them all the time?

Well, they're not well-supported by [sccache](https://lib.rs/crates/sccache), which I use all the time in CI, so it's not really an option there.

I'm not sure if there's any compelling reason not to use incremental builds all the time `locally`: I used to think more codegen units meant less optimization opportunities, but from what I read of the cargo docs, I'm not sure that's true anymore.

Generally I would advise setting i`ncremental = true` in `profile.release`, and just disabling it in CI by setting the environment variable `CARGO_INCREMENTAL` to `0`.

> What did we learn?

> Incremental builds are super useful to iterate quickly on something. That's why they're enabled by default for debug builds.

> If you make a lot of release builds locally, you may want to enable incremental builds for the release profile too.

>If the reason you make release builds locally is because you depend on crates that do gzip/bzip2/brotli/zstd decompression for example, and those are super slow in debug, you may want to set up [overrides](https://doc.rust-lang.org/cargo/reference/profiles.html#overrides) for those dependencies instead!

## Link-time optimization (LTO)

Unless you explicitly set `lto = "off"` in your profile in `Cargo.toml`, cargo performs "thin local LTO", which means it'll try to inline calls across codegen units within the same crate.

But if we want it to be able to inline calls across crates, we have to pick other types of LTO. We can do "fat" LTO, the classical method, and that one should be super expensive:

```
# in `futile/Cargo.toml`

[profile.release]
debug = 1
lto = "fat"
```

- Cold "fat" LTO release build: 3m39s (vs 2m04s thin-local LTO)
- Hot "fat" LTO release build: 2m50s (vs 1m11s thin-local LTO)

As expected, it's doing a lot more work, and so it's slower. (But with the promise of maybe generating faster code).

And we can do [thin LTO](https://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html):

```
# in `futile/Cargo.toml`

[profile.release]
debug = 1
lto = "thin"
```

- Cold thin LTO release build: 2m10s (vs 2m04s thin-local LTO)
- Hot thin LTO release build: 1m20s (vs 1m11s thin-local LTO)

And as promised, thin LTO is a lot faster than "fat" LTO.

- But we still have a block of 72s where it's doing something that's not linking, right?
- Right! We should probably do those measurements again once we've solved that.

## Rustc self-profiling

Since everything we've tried (and that usually works) has failed, it's time to go to the next level, which I didn't know existed until a couple days ago.

I said it was sorta easier to see what's going on when building, say, C/C++ code using gcc, because the steps were obvious: in a Makefile or so. But that wasn't entirely true.

Because what we have here, is the same as if we had a single `.c` source file that took over a minute to build. We know exactly what's being run, but we don't know where the compiler is spending time.

And I have no idea how easy it is to profile GCC, Clang, or MSVC, but I know it's really easy to profile rustc, with `-Z self-profile`

That's also a nightly-only option, so we'll use the same `RUSTC_BOOTSTRAP` crime. It's also a rustc flag, not a cargo flag, so instead of using `cargo build` we'll need to use `cargo rustc`.

First, let's make a cold build, because what we really want to know is what happens during a hot build:

```
cargo clean
RUSTC_BOOTSTRAP=1 cargo rustc --release
(cut)
    Finished release [optimized + debuginfo] target(s) in 2m 04s
```
And now let's change something in `main.rs` and rebuild, passing `-Z self-profile`. Because it's a rustc option, we need to pass it after `--`, which separates cargo options from rustc options:
```
$ RUSTC_BOOTSTRAP=1 cargo rustc --release -- -Z self-profile
   Compiling futile v1.9.0 (/home/amos/bearcove/futile)
    Finished release [optimized + debuginfo] target(s) in 1m 14s
```

rustc wrote a profile to disk, because we asked nicely:

```
ls -lhA *profdata
-rw-r--r--. 1 amos amos 39M Dec 30 12:52 futile-1004573.mm_profdata

```

To visualize it, we have a couple options! So we'll install a couple tools:

```
$ cargo install --git https://github.com/rust-lang/measureme crox flamegraph summarize

(cut)
     Summary Successfully installed crox, flamegraph, summarize!
```

Let's try summarize, the simplest, first:


```
summarize summarize futile-1004573.mm_profdata | head -10
+------------------------------------------------+-----------+-----------------+----------+------------+
| Item                                           | Self time | % of total time | Time     | Item count |
+------------------------------------------------+-----------+-----------------+----------+------------+
| evaluate_obligation                            | 58.01s    | 36.678          | 58.17s   | 31347      |
+------------------------------------------------+-----------+-----------------+----------+------------+
| LLVM_module_optimize_module_passes             | 23.74s    | 15.008          | 23.74s   | 16         |
+------------------------------------------------+-----------+-----------------+----------+------------+
| LLVM_lto_optimize                              | 21.09s    | 13.334          | 21.09s   | 16         |
+------------------------------------------------+-----------+-----------------+----------+------------+
| LLVM_module_codegen_emit_obj                   | 16.17s    | 10.220          | 16.17s   | 17         |
(cut)
```

> Cool bear's hot tip

Yes, the summarize command-line tools accepts subcommands, and one of them is summarize. The stutter is intentional.

Interesting! It's spending 58s in `evaluate_obligation`.

Let's try `flamegraph` next!

```
flamegraph futile-1004573.mm_profdata

```

Mhhh.. nothing?

```
ls -lhA *.svg
-rw-r--r--. 1 amos amos 28K Dec 30 12:58 rustc.svg
```
It's interactive and stuff, like you'd expect a flamegraph to be (you can hover on items to see their full names, the number and % of samples, and you can zoom around and filter stuff!), but again, all you get is a screenshot unless you go and do it yourself:

【img】

I've highlighted `evaluate_obligation`, since it was the most expensive bit reported by `summarize`.

There's a third way to visualize that information, which is the richest of them all, and it's to use `crox` to convert it to a "Chrome profile":


```
crox futile-1004573.mm_profdata

ls -lhA chrome_profiler.json
-rw-r--r--. 1 amos amos 161M Dec 30 13:12 chrome_profiler.json
```

Now we have a boatload of JSON that we can load into `chrome://tracing` (only works in Chromium-based browsers):
【img】

We see much of what we've seen in the flamegraph, but with richer info! Also, it's not the same graph at all: that one is a timeline, not a flame graph.

Flame graphs let us known about frequency, whereas timelines show what actually happened in chronological order. Which is made obvious if we search for `evaluate_obligation` to highlight it:
【img】

We can see there's a lot of `evaluate_obligation` calls, there's just one that's taking a lot longer. Which points to our issue: it's not that there's a lot of obligations to evaluate, it's that one is taking forever (over 40 seconds).

If you look closer, you can see there's actually a `link` item in there, at the top-right: it takes only 6.2 seconds. That's where the difference was made between GNU ld, LLVM lld and mold.


But... we still don't know what's happening. We know something's wrong, which is great! Or rather, we now strongly suspect we're hitting a pathological case somewhere.

What's a pathological case? It's when an algorithm is the slowest is can possibly be, due to some input.

Take insertion sort, for example: if you go to this [Sorting Algorithms Animation page](https://www.toptal.com/developers/sorting-algorithms) and click play on "Insertion", look at the "Reversed" input: see how long it takes? It has to do the maximum amount of swaps, because it's being given the worst input possible. By comparison, "Nearly sorted" finishes very quickly.

So here, we probably have an algorithm in rustc that performs reasonably well for most inputs, but our program somehow has a shape that makes that algorithm perform really poorly. This can happen when an algorithm accidentally has "quadratic complexity".

Allow me, for a minute, to get all scientific with MS Paint drawings.

Logarithmic complexity is fine, as the time increases less and less as the input size increases.

[img] O(log(n)) — logarithmic time complexity


Linear complexity is fine too, as far as I'm concerned: if you have twice as much input, it takes twice as long:
[img] O(n) — linear time complexity

But quadratic complexity, is not fine at all. Well, it might be fine for small values, but the time required shoots up real quick:

[img] O(n2) — quadratic time complexity


Okay, back to our actual build. We suspect something like that is happening... but where?

We've already said it would be a good idea to break down our big binary crate into smaller crates but... where to start? Plus, that's a lot of effort! I would really like to know what's causing it.

Luckily, there's an additional rustc option we can use there to record the arguments passed to its functions: so we'll known which obligation it's evaluating.

Let's change something else in `main.rs` and measure again:

```
RUSTC_BOOTSTRAP=1 cargo rustc --release -- -Z self-profile -Z self-profile-events=default,args

```


This time we'll do two things differently: first off, when converting the profile to the Chrome format, we'll ignore events below a certain duration: that'll make for a smaller, easier to explore profile.

But before we do, let's use another subcommand of `summarize: diff`:

```
$ summarize diff futile-1004573.mm_profdata futile-1034530.mm_profdata | head -10
+-------------------------------------------------+---------------+------------------+---------------+-------------+------------+------------+--------------+-----------------------+--------------------------+
| Item                                            | Self Time     | Self Time Change | Time          | Time Change | Item count | Cache hits | Blocked time | Incremental load time | Incremental hashing time |
+-------------------------------------------------+---------------+------------------+---------------+-------------+------------+------------+--------------+-----------------------+--------------------------+
| self_profile_alloc_query_strings                | +6.609373529s | +55952.80%       | +6.631967425s | +56144.08%  | +0         | +0         | +0ns         | +0ns                  | +0ns                     |
+-------------------------------------------------+---------------+------------------+---------------+-------------+------------+------------+--------------+-----------------------+--------------------------+
| finish_ongoing_codegen                          | -5.700469852s | -100.00%         | -5.701132355s | -100.00%    | +0         | +0         | +0ns         | +0ns                  | +0ns                     |
+-------------------------------------------------+---------------+------------------+---------------+-------------+------------+------------+--------------+-----------------------+--------------------------+
| LLVM_module_codegen_emit_obj                    | +840.972695ms | +5.20%           | +840.972695ms | +5.20%      | +0         | +0         | +0ns         | +0ns                  | +0ns                     |
+-------------------------------------------------+---------------+------------------+---------------+-------------+------------+------------+--------------+-----------------------+--------------------------+
| LLVM_lto_optimize                               | +614.923743ms | +2.92%           | +614.923743ms | +2.92%      | +0         | +0         | +0ns         | +0ns                  | +0ns                     |
(cut)

```

Well, it's not really a fair comparison since we passed different parameters to `self-profile`, and I guess that's what that result is showing us.

But it would be really useful if we had changed something meaningful about our code instead!

So, let's convert to a chrome profile, ignoring anything that takes less than, say... half a second.

```
$ crox --minimum-duration 500000 futile-1034530.mm_profdata
$ ls -lhA chrome_profiler.json
-rw-r--r--. 1 amos amos 345K Dec 30 13:48 chrome_profiler.json
```

(Yes, `--minimum-duration` takes microseconds. Yes, I had to look it up. No, I don't trust myself to make order-of-magnitude mistakes)

That's a much, much smaller profile! The second thing we're doing different is, instead of using `chrome://tracing`, we're going to use another visualizer: [Perfetto](https://ui.perfetto.dev/).

[img]

`--minimum-duration` made the profile a lot easier to read, eliminating the noise. Loading it in Perfetto made it much nicer to look at. And adding `-Z self-profile-events=default,args` means we can see which obligation it's evaluating, and it is:
```
TraitPredicate(<
  futures::future::Map<warp::hyper::Server<warp::hyper::server::conn::AddrIncoming, warp::hyper::service::make::MakeServiceFn<[closure@warp::Server<warp::log::internal::WithLog<[closure@warp::log::{closure#0}], warp::filter::or::Or<warp::filter::or::Or<warp::filter::or::Or<warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::get::{closure#0}]>::{closure#0}]>, warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filters::any::Any, warp::path::Exact<warp::path::internal::Opaque<serve::serve::{closure#0}::__StaticPath>>>, warp::path::Exact<warp::path::internal::Opaque<serve::serve::{closure#0}::__StaticPath>>>, warp::filter::FilterFn<[closure@warp::path::end::{closure#0}]>>>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:231:14: 231:92]>, warp::filter::and_then::AndThen<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::get::{closure#0}]>::{closure#0}]>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:235:19: 235:61]>>, warp::filter::and_then::AndThen<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::head::{closure#0}]>::{closure#0}]>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:239:19: 239:61]>>, warp::filter::and_then::AndThen<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::post::{closure#0}]>::{closure#0}]>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:243:19: 243:62]>>>>::bind_ephemeral<std::net::SocketAddr>::{closure#1}::{closure#0}]>>, [closure@warp::Server<warp::log::internal::WithLog<[closure@warp::log::{closure#0}], warp::filter::or::Or<warp::filter::or::Or<warp::filter::or::Or<warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::get::{closure#0}]>::{closure#0}]>, warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filters::any::Any, warp::path::Exact<warp::path::internal::Opaque<serve::serve::{closure#0}::__StaticPath>>>, warp::path::Exact<warp::path::internal::Opaque<serve::serve::{closure#0}::__StaticPath>>>, warp::filter::FilterFn<[closure@warp::path::end::{closure#0}]>>>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:231:14: 231:92]>, warp::filter::and_then::AndThen<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::get::{closure#0}]>::{closure#0}]>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:235:19: 235:61]>>, warp::filter::and_then::AndThen<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::head::{closure#0}]>::{closure#0}]>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:239:19: 239:61]>>, warp::filter::and_then::AndThen<warp::filter::and::And<warp::filter::FilterFn<[closure@warp::filters::method::method_is<[closure@warp::post::{closure#0}]>::{closure#0}]>, warp::filter::map::Map<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::and::And<warp::filter::map::Map<warp::filters::any::Any, [closure@src/serve/mod.rs:158:38: 158:66]>, warp::filter::FilterFn<[closure@warp::path::full::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::query::raw::{closure#0}], futures::future::Ready<std::result::Result<std::string::String, warp::Rejection>>>::{closure#0}]>, [closure@src/serve/mod.rs:165:26: 168:18]>>>, warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::header::optional<std::string::String>::{closure#0}], futures::future::Ready<std::result::Result<std::option::Option<std::string::String>, warp::Rejection>>>::{closure#0}]>>, warp::filter::unify::Unify<warp::filter::recover::Recover<warp::filter::and_then::AndThen<warp::filter::FilterFn<[closure@warp::filter::filter_fn_one<[closure@warp::body::body::{closure#0}], futures::future::Ready<std::result::Result<warp::hyper::Body, warp::Rejection>>>::{closure#0}]>, [closure@warp::body::bytes::{closure#0}]>, [closure@src/serve/mod.rs:174:26: 180:18]>>>, [closure@src/serve/mod.rs:184:13: 207:14]>>, [closure@src/serve/mod.rs:243:19: 243:62]>>>>::bind_ephemeral<std::net::SocketAddr>::{closure#0}]>
as
  warp::Future
>)
```

That is... big.


And that's from [warp](https://lib.rs/crates/warp), the web server framework I'm using in futile.

> What did we learn?

> rustc has a built-in profiler (`-Z self-profile`), and its output can be visualized in a multitude of ways. The [measureme](https://github.com/rust-lang/measureme) repository contains a `summarize` tool, which shows a table in the CLI, a `flamegraph` tool which generates a [flamegraph](https://www.brendangregg.com/flamegraphs.html), and `crox`, which converts the profile to Chrome's tracing format.

> Those can be viewed in `chrome://tracing` in a Chromium-based browser, on [Perfetto](https://ui.perfetto.dev/), [Speedscope](https://www.speedscope.app/), and more!

## Warp, I trusted you

...I didn't actually trust `warp` that much. It's perfectly fine! It just.. has a tendency to make for very very long types, like the one we see above. Just like [tower](https://lib.rs/crates/tower).

But I hadn't noticed those long compile times before... did something change recently?

Let's try Rust 1.54.0 just to check. Because `rustup` makes it unreasonably easy!

```
rustup install 1.54.0
cargo clean
cargo +1.54.0 build --release
(cut)
```

(Note: I had to commit a few crimes to get 1.54.0 to build my project since I've already updated a couple crates to Rust edition 2021 - the `RUSTC_BOOTSTRAP` escape hatch + manually adding `cargo-features = ["edition2021"]` did the trick).

Here are the timings:

- Cold 1.54.0 build: 1m12s (vs 2m04s for 1.57.0)
- Hot 1.54.0 build: 18.07s (vs 1m11s for 1.57.0)

Yeah. It's a pretty bad regression. And [it is known](https://github.com/seanmonstar/warp/issues/507#issuecomment-992048404).

Apparently Rust 1.58 should improve the situation. Does that mean... nightly should do better?

```
rustup install nightly
cargo clean
cargo +nightly build --release
(cut)
```

- Cold 2021-12-19 build: 1m05s (vs 2m04s for 1.57.0)
- Hot 2021-12-19 build: 15.25s (vs 1m11s for 1.57.0)

That is much, much better.

> So what, we just switch to nightly?

Well, we could! It would be as simple as pinning that nightly version in our `rust-toolchain.toml`, ie. changing:


```
[toolchain]
channel = "1.57.0"
components = [ "rustfmt", "clippy" ]
```

To:

```
[toolchain]
channel = "nightly-2021-12-19"
components = [ "rustfmt", "clippy" ]
```

But there's probably a workaround. Warp provides [Filter::boxed](https://docs.rs/warp/0.3.2/warp/trait.Filter.html#method.boxed), and boxing is a great way to hide actual types.

That will mean more heap allocations, but I think I can take the performance hit, so, let's give it a try.

Right now, the code I'm using to compose all routes looks like this:

```
    let livereload_route = method::get()
        .and(warp::path!("api" / "livereload"))
        .and(with_cx())
        .map(|cx: Context| warp::sse::reply(warp::sse::keep_alive().stream(sse_events(cx))));

    let catchall_get = method::get()
        .and(with_cx())
        .and_then(|cx: Context| cx.handle(routes::serve_get));

    let catchall_head = method::head()
        .and(with_cx())
        .and_then(|cx: Context| cx.handle(routes::serve_get));

    let catchall_post = method::post()
        .and(with_cx())
        .and_then(|cx: Context| cx.handle(routes::serve_post));

    let all_routes = livereload_route
        .or(catchall_get)
        .or(catchall_head)
        .or(catchall_post);
    let access_log = warp::filters::log::log("access");

    let addr: SocketAddr = config.address.parse()?;
    warp::serve(all_routes.with(access_log)).run(addr).await;
```

It's not very warp-y, since I do some routing in the `catchall_{get,head,post}` functions. You can see I've already been trying to stay away from large types, and kinda fighting off "the warp way". Probably a good sign that the framework just isn't for me - I'm excited to try out [axum](https://lib.rs/crates/axum).

Note that there's a bunch of methods and closures not shown here: they deal with [server-sent events](https://en.wikipedia.org/wiki/Server-sent_events), passing around a `Context` struct (a reference-counted state of the server), dealing with cookies, extracting the query string, etc.

Simply peppering a few `.boxed()...`

```
let all_routes = livereload_route
  .boxed()
  .or(catchall_get.boxed())
  .or(catchall_head.boxed())
  .or(catchall_post.boxed());
```

...brought down the build times significantly.

- Cold boxed build: 1m06s (vs 2m04s non-boxed)
- Hot boxed build: 14s (vs 1m11s non-boxed)


## Revisiting all the other changes

Now that we've got a new standard for our build times, let's try our other tricks again and see what change they make!

Because 14s is much much better than 1m11s, but it's already more than I'm comfortable with.

First, let's get a rough idea of where we're spending our time now.

```
RUSTC_BOOTSTRAP=1 cargo rustc --release -- -Z self-profile -Z self-profile-events=default,args
(cut)
crox --minimum-duration 500000 futile-1118608.mm_profdata
```

[img]

Okay! We're spending most of our time in codegen and LLVM: linking only accounts for about two seconds, if we're to trust that `link` item (and even then, it's spending time in `finish_ongoing_codegen`, which sounds like it's just waiting for the other parts to be finished).

First, instead of doing a release build, let's try making a debug build instead: this should, most significantly:
1) enable incremental builds,
2) bump codegen units from 16 to 256
3) disable optimizations.

- Cold debug build: 36.44s (vs 1m06s release)
- Hot debug build: 3.05s (vs 14s release)

That's more like it!!! Three seconds is super acceptable honestly.

Let's look where it's spending time again:

[img]

We've now reached the point where profiling is interfering with the build (making it more than 4x slower), and where a minimum duration of 500ms is hiding a lot of important information.

Let's try again with a minimum duration of 200ms:

[img]

Okay, so, still spending most of our time in codegen and LLVM.

The new iteration time to beat is still 3s.

The `dev` cargo profile defaults to `debug = true`, which is to say, `debug = 2`. Can we do better if we set it to 1?

- Cold `debug = 1` build: 33.86s (vs 36.44s `debug = 2`)
- Hot `debug = 1` build: 3.03s (vs 3.05s `debug = 2`)

Not a huge difference.

Okay well... is linking only taking ~961ms like it says it does?

Let's try GNU ld, which should be worse. (I just commented out the relevant section of my `.cargo/config.toml`):
- Cold "GNU ld" build: 41.50s (vs 36.44s "LLVM lld")
- Hot "GNU ld" build: 7.77s (vs 3.05s "LLVM lld")

Okay, lld definitely doing a bunch of work for us here. How about `mold`?

```
cargo clean
mold -run cargo build
(cut)
```

- Cold "mold" build: 37.18s (vs 36.44s "LLVM lld")
- Hot "mold" build: 2.96s (vs 3.05s "LLVM lld")

No significant difference here, there probably would be a difference if the linker was doing more work, but it seems lld is already doing great.

What if we disabled LTO entirely? Even thin-local LTO?

```
# in `futile/Cargo.toml`

[profile.dev]
lto = "off"
```

- Cold "lto = off" build: 35.96s (vs 36.44s "thin-local LTO")
- Hot "lto = off" build: 3.16s (vs 3.05s "thin-local LTO")

No significant difference here.

What else can we do? We've talked about splitting the bin crate into multiple crates, and it being a lot of work... but here's a split that's easy to do: we turn it into a `lib` crate and add a `bin` to it.

The `bin` part will simply be in charge of setting up error handling, tracing, and parsing command-line arguments, and it'll rely on the lib part for any actual work.

I won't go into too many details here, but the basic idea is to rename `src/main.rs` to `src/lib.rs`, then create `src/bin/futile.rs`, and move some code from the former to the latter.

Also, to add this to the `Cargo.toml` file:

```
[[bin]]
name = "futile"
path = "src/bin/futile.rs"
```

And now, well, it'll be hard to compare but let's try anyway:

- Cold build: 36.26s (vs Cold 36.44s "just-bin")
- Hot "changed bin" build: 2.47s (vs Hot 3.05s "just-bin")
- Hot "changed lib" build: 3.63s (vs Hot 3.05s "just-bin")


Mhh. When we change the lib, it's blocked on building the lib before it can build the bin again, which I think explains the longer times.

But when we change the bin, the whole lib doesn't need to be touched whatsoever, just linked, which explains the faster times.

And with `mold`?

- Cold build: 37.53s (vs 36.26s "LLVM lld")
- Hot "changed bin": 2.29s (vs 2.47s "LLVM lld")
- Hot "changed lib": 3.62s (vs 3.63s "LLVM lld")

Again, in this case, `mold` hardly makes a difference here: I think the linking is really cobbling together two static libraries (one for the lib, one for the bin), and a handful of dynamic libraries (libc, libm, etc.), so there's just not a lot of work to do.

## Splitting into more crates!

I gave it a couple hours, and now I have a lot of crates instead of just one!

```
# in the top-level `Cargo.toml`

[workspace]
members = [
	"crates/futile",
	"crates/futile-db",
	"crates/futile-config",
	"crates/futile-templating",
	"crates/futile-backtrace-printer",
	"crates/futile-highlight",
	"crates/futile-content",
	"crates/futile-frontmatter",
	"crates/futile-extract-text",
	"crates/futile-reading-time",
	"crates/futile-parent-path",
	"crates/futile-asset-rewriter",
	"crates/futile-friendcodes",
	"crates/futile-reddit",
	"crates/futile-patreon",
]

[profile.release]
debug = 1
```

It's not perfect — `futile-templating `for example, is one of the largest bits:

```
for i in crates/*; do echo $i $(tokei $i -o json | jq .Rust.code); done

crates/futile 1472
crates/futile-asset-rewriter 433
crates/futile-backtrace-printer 12
crates/futile-config 158
crates/futile-content 769
crates/futile-db 818
crates/futile-extract-text 26
crates/futile-friendcodes 78
crates/futile-frontmatter 105
crates/futile-highlight 533
crates/futile-parent-path 10
crates/futile-patreon 522
crates/futile-query 14
crates/futile-reading-time 6
crates/futile-reddit 69
crates/futile-templating 1513
```

And I don't love how the dependency graph looks:

```
cargo install cargo-deps

cargo deps --filter $(cargo metadata --format-version 1 | jq '.workspace_members[]' -r | cut -d ' ' -f 1 | tr '\n' ' ') | dot -Tsvg > depgraph.svg
```

[img]

But in my defense, there's definitely a way to lay that out that's less confusing.

Let's look at build times!

- Cold build time: 35.28s (vs 37.53s bin+lib crate)
- Hot build time (changed futile-extract-text): 4.22s
- Hot build time (changed futile-content): 3.97s
- Hot build time (changed futile-templating): 3.74s
- Hot build time (changed bin): 2.70s

Well. At least with incremental builds, this didn't change much.

Let's try release builds again maybe?

- Cold release build time: 1m03s (vs 1m06s)
- Hot release build time (changed futile-extract-text): 12s (vs 14s)
- Hot release build time (changed futile-content): 11.87s (vs 14s)
- Hot release build time (changed futile-templating): 11.94s (vs 14s)
- Hot release build time (changed bin): 8s (vs 14s)

That's interesting! Does linking dominate those times? I wonder. Let's try with mold again, since there's probably more linking work to be done there.

- Cold release mold: 1m04s (vs 1m06s)
- Hot release mold (futile-extract-text): 12s (vs 14s)
- Hot release mold (futile-content): 10.23s (vs 14s)
- Hot release mold (futile-templating): 11.84s (vs 14s)
- Hot release mold (bin): 8.10s (vs 14s)

Not super different from `lld`: again, I guess we don't have a lot of linking to do.

Let's look again at a rustc profile:
[img]

Mh! It seems we're spending a bunch of time "monomorphizing", which is when we turn something lik

```
fn add<T>(a: T, b: T) -> T
where
    T: Add,
{
    a + b
}
```

Into this:

```
fn add_usize(a: usize, b: usize) -> usize {
  a + b
}
```


######


Except with much bigger types. And many of these.

I'll leave you with a last tip: another factor in build time is the sheer amount of LLVM IR being generated, and you can easily look at that with cargo-llvm-lines.

For best results, we'll want to pass a couple other flags. -Zsymbol-mangling-version=v0 enables a new mangling scheme which should be the default soon, and -Z build-std instructs cargo to build libstd from source, instead of using the prebuilt version that comes with our rust distribution.

Let's try it out!

```
$ cd crates/futile
$ RUSTFLAGS=-Zsymbol-mangling-version=v0 cargo -Z build-std llvm-lines --bin futile | head -20
   Compiling futile v1.9.0 (/home/amos/bearcove/futile/crates/futile)
warning: `futile` (bin "futile") generated 2 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 6.30s
  Lines          Copies       Function name
  -----          ------       -------------
  304016 (100%)  6746 (100%)  (TOTAL)
    2665 (0.9%)     1 (0.0%)  futile[507d45e82218702e]::serve::routes::revision_routes::serve_single::{closure#0}
    2462 (0.8%)     1 (0.0%)  <futile[507d45e82218702e]::serve::Context>::serve_template::{closure#0}
    2222 (0.7%)     1 (0.0%)  <hyper[cbe7455c31603e19]::proto::h2::server::Serving<hyper[cbe7455c31603e19]::common::io::rewind::Rewind<hyper[cbe7455c31603e19]::server::tcp::addr_stream::AddrStream>, hyper[cbe7455c31603e19]::body::body::Body>>::poll_server::<hyper[cbe7455c31603e19]::service::util::ServiceFn<<warp[d2eac20ce511edd0]::server::Server<warp[d2eac20ce511edd0]::filters::log::internal::WithLog<warp[d2eac20ce511edd0]::filters::log::log::{closure#0}, warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(warp[d2eac20ce511edd0]::filters::sse::SseReply<warp[d2eac20ce511edd0]::filters::sse::SseKeepAlive<futures_util[db872dedfe3ac01e]::stream::stream::chain::Chain<futures_util[db872dedfe3ac01e]::stream::iter::Iter<alloc[f5d0236c37905b20]::vec::into_iter::IntoIter<core[49c27447ffaf9e48]::result::Result<warp[d2eac20ce511edd0]::filters::sse::Event, core[49c27447ffaf9e48]::convert::Infallible>>>, futures_util[db872dedfe3ac01e]::stream::stream::filter_map::FilterMap<tokio_stream[b4aa1729856eb832]::wrappers::broadcast::BroadcastStream<alloc[f5d0236c37905b20]::string::String>, core[49c27447ffaf9e48]::future::from_generator::GenFuture<futile[507d45e82218702e]::serve::serve::{closure#0}::sse_events::{closure#0}::{closure#0}>, futile[507d45e82218702e]::serve::serve::{closure#0}::sse_events::{closure#0}>>>>,)>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>>>>::bind_ephemeral<std[5c4e0e91f40690d7]::net::addr::SocketAddr>::{closure#1}::{closure#0}::{closure#0}, hyper[cbe7455c31603e19]::body::body::Body>, hyper[cbe7455c31603e19]::common::exec::Exec>
    2218 (0.7%)     1 (0.0%)  <hyper[cbe7455c31603e19]::proto::h2::server::H2Stream<warp[d2eac20ce511edd0]::filter::service::FilteredFuture<warp[d2eac20ce511edd0]::filters::log::internal::WithLogFuture<warp[d2eac20ce511edd0]::filters::log::log::{closure#0}, warp[d2eac20ce511edd0]::filter::or::EitherFuture<warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(warp[d2eac20ce511edd0]::filters::sse::SseReply<warp[d2eac20ce511edd0]::filters::sse::SseKeepAlive<futures_util[db872dedfe3ac01e]::stream::stream::chain::Chain<futures_util[db872dedfe3ac01e]::stream::iter::Iter<alloc[f5d0236c37905b20]::vec::into_iter::IntoIter<core[49c27447ffaf9e48]::result::Result<warp[d2eac20ce511edd0]::filters::sse::Event, core[49c27447ffaf9e48]::convert::Infallible>>>, futures_util[db872dedfe3ac01e]::stream::stream::filter_map::FilterMap<tokio_stream[b4aa1729856eb832]::wrappers::broadcast::BroadcastStream<alloc[f5d0236c37905b20]::string::String>, core[49c27447ffaf9e48]::future::from_generator::GenFuture<futile[507d45e82218702e]::serve::serve::{closure#0}::sse_events::{closure#0}::{closure#0}>, futile[507d45e82218702e]::serve::serve::{closure#0}::sse_events::{closure#0}>>>>,)>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>>>, hyper[cbe7455c31603e19]::body::body::Body>>::poll2
    2191 (0.7%)     1 (0.0%)  futile[507d45e82218702e]::serve::serve::{closure#0}
    2080 (0.7%)     1 (0.0%)  <futile_patreon[a3d798751567a674]::FutileCredentials>::load_from_cookies::{closure#0}
    1871 (0.6%)     1 (0.0%)  <futile_patreon[a3d798751567a674]::PatreonCredentials>::to_futile_credentials_once::{closure#0}
    1579 (0.5%)     1 (0.0%)  futile[507d45e82218702e]::serve::routes::revision_routes::serve::{closure#0}
    1494 (0.5%)     1 (0.0%)  futile[507d45e82218702e]::serve::routes::login::serve_patreon_oauth::{closure#0}
    1438 (0.5%)     1 (0.0%)  <hyper[cbe7455c31603e19]::proto::h2::PipeToSendStream<hyper[cbe7455c31603e19]::body::body::Body> as core[49c27447ffaf9e48]::future::future::Future>::poll
    1387 (0.5%)    94 (1.4%)  warp::server::Server<warp::filters::log::internal::WithLog<[closure@warp::filters::log::log::{closure#0}], warp::filter::or::Or<warp::filter::or::Or<warp::filter::or::Or<warp::filter::boxed::BoxedFilter<
    1329 (0.4%)     1 (0.0%)  futile[507d45e82218702e]::serve::routes::serve_get::{closure#0}
    1311 (0.4%)     1 (0.0%)  <hyper[cbe7455c31603e19]::proto::h1::dispatch::Dispatcher<hyper[cbe7455c31603e19]::proto::h1::dispatch::Server<hyper[cbe7455c31603e19]::service::util::ServiceFn<<warp[d2eac20ce511edd0]::server::Server<warp[d2eac20ce511edd0]::filters::log::internal::WithLog<warp[d2eac20ce511edd0]::filters::log::log::{closure#0}, warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::or::Or<warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(warp[d2eac20ce511edd0]::filters::sse::SseReply<warp[d2eac20ce511edd0]::filters::sse::SseKeepAlive<futures_util[db872dedfe3ac01e]::stream::stream::chain::Chain<futures_util[db872dedfe3ac01e]::stream::iter::Iter<alloc[f5d0236c37905b20]::vec::into_iter::IntoIter<core[49c27447ffaf9e48]::result::Result<warp[d2eac20ce511edd0]::filters::sse::Event, core[49c27447ffaf9e48]::convert::Infallible>>>, futures_util[db872dedfe3ac01e]::stream::stream::filter_map::FilterMap<tokio_stream[b4aa1729856eb832]::wrappers::broadcast::BroadcastStream<alloc[f5d0236c37905b20]::string::String>, core[49c27447ffaf9e48]::future::from_generator::GenFuture<futile[507d45e82218702e]::serve::serve::{closure#0}::sse_events::{closure#0}::{closure#0}>, futile[507d45e82218702e]::serve::serve::{closure#0}::sse_events::{closure#0}>>>>,)>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>, warp[d2eac20ce511edd0]::filter::boxed::BoxedFilter<(alloc[f5d0236c37905b20]::boxed::Box<dyn warp[d2eac20ce511edd0]::reply::Reply>,)>>>>>::bind_ephemeral<std[5c4e0e91f40690d7]::net::addr::SocketAddr>::{closure#1}::{closure#0}::{closure#0}, hyper[cbe7455c31603e19]::body::body::Body>, hyper[cbe7455c31603e19]::body::body::Body>, hyper[cbe7455c31603e19]::body::body::Body, hyper[cbe7455c31603e19]::server::tcp::addr_stream::AddrStream, hyper[cbe7455c31603e19]::proto::h1::role::Server>>::poll_write
    1252 (0.4%)    72 (1.1%)  warp::filters::log::log::{closure#0}], warp::filter::or::EitherFuture<warp::filter::or::Or<warp::filter::or::Or<warp::filter::boxed::BoxedFilter<
    1249 (0.4%)     1 (0.0%)  <color_backtrace[f3bfce8006cb17f7]::Frame>::print::<futile[507d45e82218702e]::serve::html_color_output::HtmlColorOutput>
    1249 (0.4%)     1 (0.0%)  <color_backtrace[f3bfce8006cb17f7]::Frame>::print::<termcolor[ed53d2579171698e]::StandardStream>
    1228 (0.4%)     1 (0.0%)  <hyper[cbe7455c31603e19]::proto::h1::io::Buffered<hyper[cbe7455c31603e19]::server::tcp::addr_stream::AddrStream, hyper[cbe7455c31603e19]::proto::h1::encode::EncodedBuf<bytes[65b0fce232ff3b0d]::bytes::Bytes>>>::parse::<hyper[cbe7455c31603e19]::proto::h1::role::Server>
    ```


    What to do with those results? Well, nothing looks out of the ordinary here. If a single symbol was responsible for say, >10% of LLVM lines (left column), we'd definitely want to take a closer look at it. But from the looks of this, we're fine.

## Conclusion

I hope you had fun learning about all this, and that you can use it to make your builds faster. If you didn't before, you should now have a lot of places to look at when you want to make your builds faster.

You can go deeper than this still! I mentioned in How I learned to love build systems that some companies don't even use cargo to build their Rust code, and I've recently been experimenting with that myself — and it's quite usable, even when you're not a megacorp.

Let me know if you'd like to read more about this: I realized that I've accidentally acquired some expertise in release engineering, so now I'm bound to get more questions about this.

Until next time, take care!
