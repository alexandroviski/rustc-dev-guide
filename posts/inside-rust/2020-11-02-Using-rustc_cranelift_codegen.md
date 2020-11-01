---
layout: post
title: "Using rustc_codegen_cranelift for debug builds"
author: Joshua Nelson
team: The Compiler Team <https://www.rust-lang.org/governance/teams/compiler>
---

## What is `rustc_codegen_cranelift`?

[`rustc_codegen_cranelift`], or just `cg_clif` for short, is a new experimental
codegen backend for the Rust compiler. The existing backend is LLVM, which very
good at producing fast, highly optimized code, but is not very good at
compiling code quickly. `cg_clif`, which uses the [Cranelift] project, would
provide a fast backend which greatly improves compile times, at the cost of
very few optimizations. This is a great fit for debug builds, and the hope is
that `cg_clif` will eventually be the default backend in debug mode.

## What is the progress of using `rustc_codegen_cranelift` for debug builds?

There has been an [Major Change Proposal][MCP] open for some time for making
`cg_clif` part of the main Rust repository. Recently, [the MCP was
accepted][compiler-team#270] and the compiler team [merged][#77975]
`rustc_cranelift_codegen` [into the main Rust git repository][#77975].
`cg_clif` still is not distributed with `rustup`, but this means you can now
build it from source in-tree!

## How do I use `rustc_codegen_cranelift`?

In this section, I'll walk through step-by-step how to build the new backend from source, then use it on your own projects. All code is copy/paste-able, and each step is explained.

First, download the Rust repository.

```console
git clone https://github.com/rust-lang/rust
```

There are a few bugs with bootstrapping that were only recently fixed and haven't yet been merged, so we need to use the changes from [#78624].

[#78624]: https://github.com/rust-lang/rust/pull/78624

```sh
$ git fetch origin pull/78624/head
$ git checkout FETCH_HEAD
```

Now, let's set up the build system to use `cg_clif`.

```toml
$ cat > config.toml <<EOF
[rust]
codegen-backends = ["cranelift"]
EOF
```

Finally, let's run the build. This can take a long time, over a half-hour in some cases.

```console
./x.py build
```

We now have a working rust compiler using `cg_clif`!
We can make it easy to use with `rustup toolchain link`, as explain in the [`rustc-dev-guide`]:

```sh
# Replace <host-triple> as appropriate.
$ rustup toolchain link cg_clif build/<host-triple>/stage1
```

Finally, we can start using it for our own projects. For demonstration
purposes, I'll be be using my `saltwater` project, but you can use any Rust
project supported by `cg_clif`.

```
$ git clone https://github.com/jyn514/saltwater
$ cd saltwater
$ cargo +cg_clif build
   Compiling saltwater-parser v0.11.0 (/home/joshua/saltwater/saltwater-parser)
   Compiling saltwater-codegen v0.11.0 (/home/joshua/saltwater/saltwater-codegen)
   Compiling saltwater v0.11.0 (/home/joshua/saltwater)
    Finished dev [unoptimized + debuginfo] target(s) in 23.86s
```

It works! For comparison, let's see how long the equivalent LLVM backend would
take. `nightly` includes various optimizations we didn't run locally, so let's
build the LLVM backend from source to have a good comparison. To avoid removing
all the hard work we did to build cranelift, we'll add a new [git worktree] with
a separate build cache.

```sh
$ cd rustc
$ git worktree add ../rustc-llvm
$ cd ../rustc-llvm
# Avoid building llvm from source if possible
$ printf '[llvm]\ndownload-ci-llvm = "if-available"' > config.toml
# Again, this may take a very long time.
$ x.py build
$ rustup toolchain link llvm build/<host-triple>/stage1
```

Now let's try building `saltwater` again:

```
$ cd saltwater
$ cargo +llvm build
    Finished dev [unoptimized + debuginfo] target(s) in 28.77s
```

LLVM takes a full 5 seconds longer for a full build.

## How can I help?

You don't need to be a compiler developer to help improve `cg_clif`!  The best
way you can help is by testing `cg_clif` on different Rust crates across the
ecosystem.  Just while writing this article, I found [two][#1102]
[bugs][#1101], so there's plenty of work left to be done. Please report any bugs you find
to the [`rustc_codegen_cranelift` git repository][issue].

In the future, we hope to distribute `cg_clif` with Rustup, and if it matures sufficiently, eventually make it the default backend for debug builds.

[`rustc_codegen_cranelift`]: https://github.com/bjorn3/rustc_codegen_cranelift
[Cranelift]: https://github.com/bytecodealliance/wasmtime/tree/main/cranelift#cranelift-code-generator
[#77975]: https://github.com/rust-lang/rust/pull/77975
[MCP]: https://forge.rust-lang.org/compiler/mcp.html
[compiler-team#270]: https://github.com/rust-lang/compiler-team/issues/270
[`rustc-dev-guide`]: https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html#creating-a-rustup-toolchain
[git worktree]: https://rustc-dev-guide.rust-lang.org/building/suggested.html#working-on-multiple-branches-at-the-same-time
[#1102]: https://github.com/bjorn3/rustc_codegen_cranelift/issues/1102
[#1101]: https://github.com/bjorn3/rustc_codegen_cranelift/issues/1101
[issue]: https://github.com/bjorn3/rustc_codegen_cranelift/issues/new
