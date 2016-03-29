- Feature Name: N/A
- Start Date: 2016-03-28
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add the ability for crates to run build scripts from other crates without
writing a build script in the dependency.

# Motivation
[motivation]: #motivation

With the support added in [RFC 1573] the ergnomics of code generation should be
greatly improved. With nicer error messages more naturally written code we'd
make great strides towards operating basically the same as the built-in
`#[derive]` (but more powerfully).

[RFC 1573]: https://github.com/rust-lang/rfcs/pull/1573

This support, however, still requires crates using code generation to write a
build script. Namely, each crate must perform the following *extra* steps to
leverage code generation:

1. A build script is specified in `Cargo.toml` via `build = "build.rs"`
2. A build dependency is added to `Cargo.toml`, typically with the code
   generation of the crate in question. For example:

   ```toml
   [build-dependencies]
   serde-codegen = "0.5"
   ```

3. The build script is filled out, typically with a "one liner" like:

   ```rust
   extern crate serde_codegen;

   fn main() {
       serde_codegen::process();
   }
   ```

While not the highest bar to entry ever, it's clear that this is less ergnomic
than the alternative of doing nothing at all! The purpose of this RFC is to set
forth a vision and implementation to move even closer to this "zero overhead"
custom derive mode. The driving goal is the almost nothing is written down in
`Cargo.toml`.

# Detailed design
[design]: #detailed-design

The syntax for depending on crates in `Cargo.toml` will be extended to allow for
a binary to be run as a build script from dependencies, for example:

```toml
[build-dependencies]
serde-codegen = { version = "0.5", build-script = "process" }
```

This syntax means that the `serde-codegen` crate is dependend upon and it will
run the binary `process` from `serde-codegen` for this crate. The `build-script`
key will only be allowed in the `[build-dependencies]` section, and any number
of scripts can be specified. Local build scripts *cannot depend on
`serde-codegen`* as it won't be available as a library, just built as a binary
somewhere. Build scripts from other crates will run after the local crate's
build script, and they will run in the order specified in `Cargo.toml` (top to
bottom).

On the "producer side" crates which want to export a binary to be runnable as a
build script would use the syntax:

```toml
[[bin]]
name = "process"
build-script = true
```

This whitelist ensures that assorted binaries that are members of a project
aren't considered as part of the public interface of a library unless explicitly
stated.

### Interface for build scripts

Each build script will largely run with [the same inputs][bs-inputs] that build
scripts have today. One notable difference is that the `CARGO_FEATURE_*`
environment variables will be relative to the crate defining the build script
rather than the target crate itself.

In order to support composing multiple build scripts the output of one run needs
to be available as the input to another run. Cargo will prepare an input and
output directory for each build script being run. The first input is the
original source directory, and each subsequent input is the output of the
previous run. After a build script has finished, Cargo will overlay all
other input files from the original source into output directory that don't
already exist. Source maps proposed in [RFC 1573] will be used to ensure that
errors work out.

[bs-inputs]: http://doc.crates.io/build-script.html#inputs-to-the-build-script

### Example build flow

Let's walk through an example of a source a few files. The `src/lib.rs` entry
point depends on `src/foo.rs` which in turn has `#[derive(Deserialize)]` as well
as an inline `bindgen!` macro to expand to a bunch of FFI items. To start off,
let's take a look at our `Cargo.toml`

```toml
[package]
name = "rfc-example"
version = "0.1.0"
authors = ["..."]

[dependencies]
serde = "0.7"

[build-dependencies]
serde-codegen = { version = "0.7", build-script = "process" }
bindgen = { version = "0.3", build-script = "process" }
```

Here we depend on `serde` for runtime support and `serde-codegen` for expanding
the `#[derive(Deserialize)]` attribute. The `bindgen` crate is also depended on
to expand our `bindgen!` macro.

When Cargo builds this crate, it will first run the `serde-codegen` build script
(as it's listed first). The input directory will be the checked out source, and
the output directory (`OUT_DIR`) will be a blank directory created by Cargo. The
`process` binary from `serde-codegen` will then create and output directory that
looks like:

```
$OUT_DIR/src/foo.rs
$OUT_DIR/src/foo.rs.map
```

Cargo then knows that there is another build script to run, `process` from
`bindgen`, so it fills in the rest of the source tree:

```
$OUT_DIR/src/foo.rs
$OUT_DIR/src/foo.rs.map
$OUT_DIR/src/lib.rs       (copied/linked from original src/lib.rs)
$OUT_DIR/src/lib.rs.map   (indicates original source was src/lib.rs)
```

Note that Cargo does not overwrite `src/foo.rs` with the original copy as it's
assumed that the source was preserved. Next, cargo prepares a second output
directory, `$OUT_DIR2`, which it then runs `bindgen` over, creating the output:

```
$OUT_DIR2/src/foo.rs
$OUT_DIR2/src/foo.rs.map
```

Finally, Cargo will run the compiler with an input file as `$OUT_DIR/src/lib.rs`
and an include path of `$OUT_DIR2`.

# Drawbacks
[drawbacks]: #drawbacks

* This is a relatively weighty feature to be added for only removing a few lines
  from `Cargo.toml` and a build script, and this arguably may not be worth the
  ergonomics that are regained. Additionally, this still isn't as "sweet" as
  depending on `rustc-serialize` because to get full Serde support you need two
  dependencies in different sections.

* Extending build scripts further and broadening the use cases for them pushes
  the ecosystem farther away from ever using a non-Cargo-based build system, and
  can make efforts to integrate Cargo into other build systems more difficult in
  the future.

# Alternatives
[alternatives]: #alternatives

* Cargo could forgo much of the complications here by only allowing one build
  script per crate. In that scenario there's no worry about composing scripts as
  there is no compositionality. This unfortunately means that depending on
  serde + bindgen forces you to write a build script, and may end up being a
  common scenario.

* Instead of requiring `build-script = "process"` keys, crates could instead
  declare themselves as build scripts (like plugins do). This would mean that
  adding `serde-codegen` to a `[dependencies]` section would be an error (as it
  would not provide a library), but it would not be clear which build
  dependencies are used by the local build script and which are preprocessors.

# Unresolved questions
[unresolved]: #unresolved-questions

* The pipelining and composing of build scripts together is pretty complicated
  and has unclear performance characteristics. Can this be implemented in an
  efficient fashion use clever amounts of symlinks, hard links, junctions, etc?
