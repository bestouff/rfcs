- Feature Name: N/A
- Start Date: 2015-09-15
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Improve Cargo's story around multi-crate single-repo project management by
sharing `Cargo.lock` among crates and sharing an output directory for artifacts
by default.

Cargo will infer connections between crates where possible, but it will also
have knobs for explicitly controlling relationships among crates.

# Motivation

A common method to organize a multi-crate project is to have one
repository which contains all of the crates. Each crate has a corresponding
subdirectory along with a `Cargo.toml` describing how to build it. There are a
number of downsides to this approach, however:

* Each sub-crate will have its own `Cargo.lock`, so it's difficult to ensure
  that the entire project is using the same version of all dependencies. This is
  desired as the main crate (often a binary) is often the one that has the
  `Cargo.lock` "which counts", but it needs to be kept in sync with all
  dependencies.

* When building or testing sub-crates, all dependencies will be recompiled as
  the target directory will be changing as you move around the source tree. This
  can be overridden with `build.target-dir` or `CARGO_TARGET_DIR`, but this
  isn't always convenient to set.

Solving these two problems should help ease the development of large Rust
projects by ensuring that all dependencies remain in sync and builds by default
use already-built artifacts if available.

# Detailed design

Cargo will grow the concept of a **workspace** for managing repositories of
multiple crates. Workspaces will then have the properties:

* A workspace can contain multiple local crates.
* Each workspace will have one root crate.
* Whenever any crate in the workspace is compiled, output will be placed in the
  `target` directory next to the root crate.
* One `Cargo.lock` for the entire workspace will reside next to the root crate
  and encompass the dependencies (and dev-dependencies) for all packages in the
  workspace.

With workspaces, Cargo can now solve the problems set forth in the motivation
section. Next, however, workspaces need to be defined. In the spirit of much of
the rest of Cargo's configuration today this will largely be automatic for
conventional project layouts but will have explicit controls for configuration.

### New manifest keys

First, let's look at the new manifest keys which will be added to `Cargo.toml`:

```toml
[package]

# ...

workspace-root = true
workspace = ["relative/path/to/child1", "child2"]
```

Here the `workspace-root` key will be used to indicate whether a package is the
root of a workspace, and the `workspace` key will be a list of paths to crates
which should be added to the package's workspace. The paths listed in
`workspace` must be valid paths to crates.

### Implicit relations

In addition to the keys above, Cargo will apply a few heuristics to infer the
keys wherever possible:

* All path dependencies of a crate are considered members of the `workspace` key
  implicitly.
* Starting from a package's `Cargo.toml`, Cargo will walk upwards on the
  filesystem to find a sibling `Cargo.toml` and VCS directory (e.g. `.git` or
  `.svn`). If found, this crate is also implicitly considered a member of the
  workspace.
* Crates whose `Cargo.toml` that reside next to VCS directories are implicitly
  workspace roots.

These rules are intended to reflect conventional Cargo project layouts. "Root
crates" typically appear at the root of a repository with lots path dependencies
to all other crates in a repo. Additionally, we don't want to traverse wildly
across the filesystem so we only go upwards to a fixed point or downwards to
specific locations.

### Constructing a workspace

With the explicit and implicit relations defined above, each crate will now have
a flag indicating whether it's the root and a number of outgoing edges to other
crates. Two crates are then in the same workspace if they both transitively have
edges to one another. A valid workspace then only has one crate that is a root.

While the restriction of one-root-per workspace may make sense, the restriction
of crates transitively having edges to one another may seem a bit odd. The
intention is to ensure that the set of packages in a workspace is the same
regardless of which package is selected to start discovering a workspace from.

With the implicit relations defined it's possible for a repository to not have a
root package yet still have path dependencies. In this situation each dependency
would not know how to get back to the "root package", so the workspace from the
point of view of the path dependencies would be different than that of the root
package. This could in turn lead to `Cargo.lock` getting out of sync.

To alleviate misconfiguration, however, if the `workspace` configuration key
contains a crate which is not a member of the constructed workspace, Cargo will
emit an error indicating such.

### Workspaces in practice

It's expected that most of the time repositories need no modification to benefit
from workspaces. If a root package exists and transitively depends on all other
crates in a repository, then no configuration is necessary to construct the
workspace.

Projects like the compiler, however, will likely need explicit configuration.
The `rust` repo conceptually has two workspaces, the standard library and the
compiler, and these would need to be manually configured with `workspace` and
`workspace-root` keys amongst all crates.

# Drawbacks

* This change is not backwards compatible with older versions of Cargo.lock. For
  example if a newer cargo were used to develop a repository which otherwise is
  developed with older versions of Cargo, the `Cargo.lock` files generated would
  be incompatible. If all maintainers agree on versions of Cargo, however, this
  is not a problem.

* Given the heuristics above, there's a bit of a "configuration cliff" where if
  you don't fall into the conventional structure then *all* crates may need
  configuration to construct a workspace.

* As proposed there is no method to disable implicit actions taken by Cargo.
  It's unclear what the use case for this is, but it could in theory arise.

# Alternatives

* Cargo could attempt to perform more inference of workspace members by simply
  walking the entire directory tree starting at `Cargo.toml`. All children found
  could implicitly be members of the workspace. Walking entire trees,
  unfortunately, isn't always efficient to do and it would be unfortunate to
  have to unconditionally do this.

* Implicit members are currently only path dependencies and a "Cargo.toml next
  to VCS" traveling upwards. Instead all Cargo.toml members found traveling
  upwards could be implicit members of a workspace. This behavior, however, may
  end up picking up too many crates.

# Unresolved questions

* Does this approach scale well to repositories with a large number of crates?
  For example does the winapi-rs repository experience a slowdown on standard
  `cargo build` as a result?
