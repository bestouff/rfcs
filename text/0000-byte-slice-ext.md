- Feature Name: `byte_slice_ext`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Deprecate the `std::slice::bytes` module with a new `ByteSliceExt` trait to
reside in the `std::slice` module, enhancing `[u8]` as a type.

# Motivation

Currently the methods in `std::slice::bytes` are useful for moving around bytes
and calling efficient intrinsics such as `memcpy` or `memset`, but the methods
are unstable. The purpose of this RFC is to provide a path to stabilization for
these methods.

# Detailed design

The entire `std::slice::bytes` module will be `#[deprecated]` and will be
replaced with the following trait to reside in the `std::slice` module.

```rust
trait ByteSliceExt: SliceExt<Item=u8> {
    /// Fill this entire slice with the specified byte. Safe variant of
    /// ptr::write_bytes
    fn write_bytes(&mut self, byte: u8);

    /// Copy the entire contents of this slice into the destination.
    ///
    /// Panics if the destination's length is shorter than this slice.
    fn copy_to(&self, dst: &mut [u8])
}

impl ByteSliceExt for [u8] { ... }
```

# Drawbacks

* If this trait were to be generically expanded for other byte-like types (e.g.
  `&[u16]`) then the name `ByteSliceExt` doesn't quite fit the bill. It is
  believed, however, that there would be a greater benefit from another piece of
  functionality to convert `&[u16]` to `&[u8]`, however.

* The design is currently quite conservative. For example there are efficient
  implementations for operations such as "find a byte in a slice" or "find a
  slice in a slice" available in libc. It is unclear, however, whether these
  optimized implementations should be exposed as methods on this trait or
  through another means, such as specialization.

# Alternatives

* Instead of creating a new `ByteSliceExt` trait, the stabilization of these
  methods could be postponed until equality constraints in where clauses are
  supported. Each method could then be added to the `SliceExt` trait directly
  with a clause of `where Self::Item = u8`. This means that byte slices are
  "magically enhanced" with more methods by default (due to the prelude position
  of `SliceExt`) when compared with other slices.

* The `ByteSliceExt` trait could be placed in the prelude, but the drawback
  listed above applies to this alternative as well.

* Instead of hardwiring to `u8`, instead provide an implementation for `[i8]` as
  well.

# Unresolved questions

None so far.
