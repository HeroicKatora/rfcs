- Feature Name: (`transparent-patterns`)
- Start Date: (2019-05-21)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow patterns to denote a structure annotated with `#[repr(transparent)]` in
place of the structural pattern for the basis of the representation.

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
* Avoids one level of indirection and a lifetime. Current code [many][EX1]
  [times][EX2] [provides][EX3] [wrappers][EX4] some with [generic `impl
  AsRef<[T]>`][EX5] instead of encapsulating the actual memory region.
* Ascribing additional meaning to raw data:
  ```
  struct RGB([u8; 3]);
  ```

Calling a `&self` method on a type wrapping a reference to unsized data
suddenly has two lifetimes to care about. The one of the container struct which
is just `struct _ { inner: C, }` and the actually relevant one of the memory.
This makes it unecessarily unergonomic to store a borrowed result from such a
type.

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
mismatches. However the reverse, converting to a reference of a contained fied,
is easy since it is a basic field access. The layout of value denoted by the
inner field is afterall ensured already through the usual means (disregarding
`#[packed]`).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Usually types can be constructed and destructed via their struct syntax:

```
struct Foo { bar: usize }

let foo = Foo { bar: 0 };
let Foo { bar, } = foo;
```

This proposes to allow a pattern to match as a another type if that type is a
transparent wrapper, and *only* then. This requires the member to be accessible
and valid in that context like a normal constructor would.

```
#[repr(transparent)]
struct ascii([u8]);

let val as ascii(val) = byte_slice;
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The layout of a wrapper type annotated as `#[repr(transparent)]` is guaranteed
to be the one of the sole underlying type. Hence, it must contain exactly one
non-zero-sized field.

Allow `ref` and `ref mut` *identifier patterns* for the type of the underlying
type to contain an optional suffix `as <construction>` denoting the
construction of the wrapping type. This can be used at the same time as the
existing `@ pattern` suffix for identifier patterns but must occur before it.

```
#[repr(transparent)]
struct asii([u8]);

let val as ascii(val) = byte_slice;
// Desugared:
let &ref val as ascii(val) = byte_slice;

#[repr(transparent)]
struct AsciiChar(u8);

impl AsciiChar {
    fn as_ascii_char(ch: &u8) -> Option<&AsciiChar> {
        match ch {
            ch as AsciiChar(ch) @ 0..=127 => Some(ch),
            _ => None,
        }
    }
}
```

Conceptually, the optional `as <construction>` argument allows viewing of the
matched *place*, optionally including multiple zero-sized slices at eiter end,
*without permitting reinterpretation* since the memory contain the type itself
will remain being interpreted as that type including its invariants.

The construction expression itself must consist only of:

* exactly one *struct expression*
* where exactly the wrapped field names the identifier
* and all other fields name a zero sized type

```
#[repr(transparent)] 
struct unparsed<T> {
    data: [u8],
    to_be_type: PhantomData<T>,
};

// Ok
let data as unparsed {
    data,
    to_be_type: PhantomData::<String>,
} = raw_bytes;

fn produces() -> PhantomData<String> { .. }

// NOT Ok
let data as unparsed {
    data,
    to_be_type: produces(), // No full expressions.
} = raw_bytes;

// NOT Ok
let data as unparsed {
    data: &data[1..], // No expressions on the value.
    to_be_type: PhantomData::<String>,
} = raw_bytes;

// Ok
let val as unparsed {
    data: val, // Can give it a different name.
    to_be_type: PhantomData::<String>,
} = raw_bytes;
```

# Drawbacks
[drawbacks]: #drawbacks

This complicates pattern matching.

It introduces a value-like syntax to places where actually no values are
involved.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Non-`ref` patterns are explicitely excluded to not further dilute the meaning
of patterns not to involve expression evaluation on their own.

An alternative is embracing unsized values. This would solve the problem of
construction but does not address efficiency and usability concerns. In
particular, it would not allow converting the type of values already behind a
reference.

# Prior art
[prior-art]: #prior-art

C permits casting of types that have 'compatible layout' but the compiler
undertakes no effort of validating it, silently permitting undefined behaviour
to occur.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should a slice of the wrapper type also be constructible from a slice of the
wrapped type? This seems useful in avoiding duplicating every definition for a
slice variant. But for unsized original types or other usecases this is not
even be applicable so that further syntactical or semantical complications
would be suspect.

# Future possibilities
[future-possibilities]: #future-possibilities

Leverage the concept for the benefit of several

Other types and representations also permit transmutes of effective transmutes
e.g. to `[u8]` by having no alignment holes in their layout such as integer
types. For some applications, this is not a problem since those types are
usually `Copy` but does not hold in general. Support methods such as
`u16::to_{ne,be,le}_bytes` further increase usability but do not establish a
general pattern: Their internal implementations are still a standard
`transmute` and so does not naturally expand to slices.

```
let slice as <[u8]> = u16_slice;
let [upper, lower] = &mut u16_value;
```

But note that this is related but different direction: This RFC is focussed on
*construction* of a wrapper that enables additional invariants while this other
possibility would be *destructuring*.
