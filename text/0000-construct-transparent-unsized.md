- Feature Name: (`construct_unsized_transparent_wrapper`)
- Start Date: (fill me in with today's date, 2020-10-24)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Provide a way to construct transparent wrappers from a reference to the
wrapped, non-zero size field.

# Motivation
[motivation]: #motivation

Custom dynamically sized types (wrapping one of the native DSTs) are in an
unfortunate place right now. It is almost entirely impossible to create them
within a safe context and since they exist only (maybe soon mostly) behind
references, `#[repr(transparent)]` is also mostly useless for this job. Custom 
newtype wrappers around usual types are also not optimal when interacting 
with code that instead targets the underlying type.

However, being able to have new invariants on (unsized) types without
changing representation has many real advantages:
* `str` can be regarded as such. While an internal type for the moment (and
  likely longer), a custom type in its likeness for other encodings can be
  useful.
* Network programming and other forms of communication *rarely* deal with fixed
  size structures but have highly predictable content and internal invariants.
* `[impl Ord]` that actually maintains order in the slice.
* Avoids one level of indirection and a lifetime. [Current][EX4] code [many][EX1]
  times [provides][EX2] wrappers [some][EX3] with [generic `impl
  AsRef<[T]>`][EX5] instead of encapsulating the actual memory region.
* Ascribing additional meaning to raw data:
  ```
  struct RGB([u8; 3]);
  ```

Calling a `&self` method on a type wrapping a reference to unsized data suddenly has two lifetimes to care
about. The one of the container struct which is just `struct _ { inner: C, }`
and the actually relevant one of the memory. This makes it unecessarily
unergonomic to store a borrowed result from such a type.

It also ties two separate concepts strongly together. The struct for
encapsulating an inner invariant is many times also in charge of owning that
data. This is however unecessary, especially but not only if the data need only
be accessed immutably but at that point library authors opt to introduce
illusive bounds of `C: AsRef<[T]> + AsMut<[T]>` instead. This creates new
problems if they actually do care about the memory location since those two
methods need not return the 'same' slices at all times. And note how the
`AsMut<[T]>` bound always conditionally pops up despite the receiving method
already declaring itself to be `&mut self`.

A much more preferable solution would be to be able to provide a native,
standard internal type on which such guarantees can be made and which will
also produce the desired (co-)variance–`&'_ T`. Note that the standard library
already follows this pattern! `Vec<T>` is the owning abstraction while `[T]` is
the representational. And there are plenty other owners of `[T]` suggesting
that indeed these concepts should be addressed in different levels of
abstraction. Similar follows for `String` and `str` where we have the
additional accommodation of being enabled to convert between `[u8]` and `str`
due to internal, unsafe magics. (Pretty literally currently since they are
`lang` items. Some code casts pointers, some does union access, it's a bit all
over the place).

[EX1]: https://docs.rs/percent-encoding/1.0.1/percent_encoding/struct.PercentEncode.html
[EX2]: https://docs.rs/regex/1.1.6/src/regex/re_unicode.rs.html#1163
[EX3]: https://doc.rust-lang.org/std/io/struct.IoVec.html
[EX4]: https://docs.rs/unicase/2.4.0/unicase/struct.UniCase.html
[EX5]: https://docs.rs/smoltcp/0.5.0/smoltcp/wire/struct.EthernetFrame.html

The type of a value can change through coercion or an explicit constructor. The
type itself can only be changed within an expression and can not be changed
when matching a reference although matching by value may instead produce
multiple values of other types. This presents a problem for dynamically sized
types which are exclusively visible behind references (and pointers). Creating
a reference to a custom DST thus always involves `unsafe` code for

1. Converting the original reference to a pointer.
1. Casting a pointer to the own DST
3. `unsafe`: Dereferencing that pointer and reborrowing it

This is error prone since the lifetime information is lost; and the involved
types may change without compilation errors introducing misalignment or size
mismatches. Using a `transmute` of the reference directly can have even larger
impact without failing to compile, if any of the involved types changes. Not 
requiring `unsafe` at this point would also encourage using it more sparingly,
improve code quality, and allow more `#[deny(unsafe)]` usage. Note that the reverse,
converting to a reference of a contained fied, is easy since it is a basic field access.
The layout of value denoted by the inner field is afterall ensured already through the 
usual means, and so is the reference creation to it (disregarding`#[packed]`).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Usually types can be constructed and destructed via their struct syntax:

```rust
struct Foo { bar: usize }

let foo = Foo { bar: 0 };
let Foo { bar, } = foo;
```

This does not usually work for unsized types since fields are values and values
need to be Sized. But when we immediately reference the value and the
constructed type is a transparent wrapper then of course no such value need to
be moved and constructed at all, it only borrows the original value wrapped in
a new type. Thus, in a context where the constructed value is immediately
borrowed we add a marker to initialize the field from a place expression
without moving the value from it.

```rust
#[repr(transparent)]
struct ascii { inner: [u8] };

let byte_slice: &[u8] = ...;
let val: &ascii = &ascii {
    #![repr(transparent)] inner: *byte_slice
};
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A struct-expression annotated with the `repr(transparent)` attribute must occur
within a reference expression and its type itself must also be annotated with
the same attribute. Its single non-zero sized field is initialized by a place
expression instead of a value expression. This has the effect of re-borrowing
the place of the field with a pointer cast.

For readability purposes the author suggests using an inner attribute instead.

```rust
#[repr(transparent)]
struct ascii([u8]);

let byte_slice: &[u8] = ...;
let val: &ascii = &ascii(#![repr(transparent)] *byte_slice);
// Equivalent:
let val: &ascii = &#[repr(transparent)] ascii(*byte_slice);
```

The reference expression around the struct expression determines the kind of
borrowing that takes place.

```rust
#[repr(transparent)]
struct asii([u8]);

let byte_slice: &mut [u8] = ...;
let val: &mut ascii = &mut ascii(#![repr(transparent)] *byte_slice);
let _ = byte_slice[0]; 
//      ^ ERROR: cannot use `byte_slice[_]` because it was mutably borrowed
// some later use of `val`.

let byte_slice: &[u8] = ...;
let val: &mut ascii = &mut ascii(#![repr(transparent)] *byte_slice);
//  ERROR: can not borrow data in a `&_` reference mutably.
```

The field does not need to be a dynamically sized type. For example:

```rust
#[repr(transparent)]
struct AsciiChar(u8);

impl AsciiChar {
    fn as_ascii_char(ch: &u8) -> Option<&AsciiChar> {
        match ch {
            ch @ 0..=127 => Some(&AsciiChar(#![repr(transparent)] *ch)),
            _ => None,
        }
    }
}
```

A transparent type can contain any number of 1-aligned, zero-sized fields.
These must also be initialized by value. For example, here we have a wrapper
around a slice that enforce that elements are sorted according to some
(statically known) comparison function.

```rust
trait Order<T> {
   const PARTIAL_CMP: fn(&T, &T) -> Option<Ordering>;
}

struct Sorted<T, U: Order<T>> {
  elements: [T],
  sorter: PhantomData<U>,
}

impl<T, U: Order<T>> Sorted<T, U> {
  pub fn new(slice: &[T]) -> Option<&Self> {
    if slice.iter().is_sorted_by(U::PARTIAL_CMP) {
      Some(&Sorted {
        #![repr(transparent)]
        elements: *slice,
        sorter: PhantomData,
      })
    } else {
      None
    }
  }
}
```

In MIR the initialization of zero-sized fields need not be translated. Instead
this will be the same as the current pointer cast.

```rust
#[repr(transparent)]
struct asii([u8]);

let byte_slice: &mut [u8] = ...;
let val: &mut ascii = &mut ascii(#![repr(transparent)] *byte_slice);
// Semantically equivalent unsafe solution:
let val: &mut ascii = unsafe { &mut *(byte_slice as *mut _ as *mut ascii) };
```

However, we still require an `unsafe` block when the borrow of place expression
itself would require one. That also improves the safety of that initialization
since changing the input type from `&[u8]` to `*const [u8]` will error until an
explicit unsafe block is put in place. In that regard, a more faithful
translation of the expression might be:

```rust
let byte_ptr: *mut [u8] = ...;
let val: &mut ascii = &mut ascii(#![repr(transparent)] *byte_ptr);

let val: &mut ascii = {
    let _ = &mut *byte_ptr; // Safety check.
// ERROR: dereference of raw pointer is unsafe and requires unsafe function or block
    unsafe { &mut *(byte_ptr as *mut _ as *mut ascii) }
};

// Fine:
let val: &mut ascii = unsafe { &mut ascii(#![repr(transparent)] *byte_ptr) };
```

# Drawbacks
[drawbacks]: #drawbacks

It uses an attribute to manipulate the meaning of a particular syntax which
complicates handling of particular expressions.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could also imagine a syntax where a field initializer that is a place
expression is treated separately and its semantics depend on the context of the
struct expression.

```rust
let byte_slice: &[u8] = ...;
let val: &ascii = &ascii(*byte_slice);
// OR
let val: &ascii = &*ascii(byte_slice);
```

However this comes with additional compatibility risks. In cases where the
wrapped type is _not_ a dynamically sized type this is already permitted syntax
that compiles when the type is `Copy`. Permitting this would impact readability
as the lifetime and borrows of the reference differ. The currently allowed case
introduces a temporary and references it while the new functionality would
borrow the field. The extra verbosity to distinguish these two cases seem fine
in particular since it's not expected to appear very often, mainly in a small
amount of wrapper constructors.

Another alternative was discussed [in a thread on internals][internals-10229]
using `as` and patterns instead. However, this is a poor match as it extends
patterns by quite a bit while wanting to express an expression. As such it
composes poorly and interacts with privacy in new ways.

[internals-10229]: https://internals.rust-lang.org/t/pre-rfc-patterns-allowing-transparent-wrapper-types/10229

```rust
#[repr(transparent)]
struct ascii([u8]);

let byte_slice: &[u8] = ...;
let (val as ascii(val)): &ascii = byte_slice;
```

# Prior art
[prior-art]: #prior-art

C permits casting of types that have ‘compatible layout’ but the compiler
undertakes no effort of validating it, silently permitting undefined behaviour
to occur. There is also no concept of privacy for the purpose of safety
invariants.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should a slice of the wrapper type also be constructible from a slice of the
wrapped type? This seems useful in avoiding duplicating every definition for a
slice variant. But for unsized original types or other use cases this is not
even be applicable so that further syntactical or semantical complications
would be suspect.

# Future possibilities
[future-possibilities]: #future-possibilities

This does not address all 'simple' wrappers. For example, a wrapper that
requires its inner field to be aligned more strictly can not be annotated
`repr(transparent)` as it already uses `repr(align)`. It would not be safe to
construct it but it could be seen as a pendant to `repr(packed)`. The latter
makes _accessing a field by reference_ unsafe while the former makes
_constructing from a reference_ unsafe. Maybe this should be permitted with a
similar syntax.
