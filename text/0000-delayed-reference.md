- Feature Name: delayed_reference
- Start Date: 2019-01-20
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Introduce an additional unnamed type and a corresponding conversion operation
to references that allows explain the current behaviour of 'borrow-as-ptr'
expressions such as `&(*foo).field as *const _`. Also, the resulting MIR will
not exhibit undefined behaviour. That is, it retroactively makes typical
statements defined even for packed structs and requires less unsafe.

# Motivation
[motivation]: #motivation

In current Rust semantics to obtain a pointer to some place, one creates a
reference and casts it to a pointer. Since we strictly speaking attach even to
this temporary reference some additional invariants, this is not strictly sound.
These invariants (among them alignedness and dereferencability) are not required
for pointers. Since any inspection of a type's fields involves an intermediate
reference to the extracted field, it is impossible to soundly retrieve a pointer
to such.

A [similarly motivated RFC](https://github.com/rust-lang/rfcs/pull/2582) exists
that tries to approach this problem by adding an operation that performs a
direct pointer to pointer-to-field conversion in one special syntactical case.
This has the immediate drawback of leaving some existing code at a state of
undefined behaviour and introducing UB interact with no-op pointer casts:

```
// UB, intermediate & is not allowed.
&packed.field as &T as *const T;
```

This proposal takes a different approach: Add MIR types and structure so that
the creation of an actual reference is delayed up to a reasonable point that
allows casting to a pointer before the reference invariants come into play.

Another shortcoming of current syntax is that dereferencing a raw pointer always
requires an `unsafe` block, even when only address calculation to another
pointer is performed. This is inconvenient and makes `unsafe` usage seem
acceptable even in a situation where programmers provide no justification for
such usage (there are no safety preconditions to such address calculation). When
another library is used to safely initialize such a raw reference, no unsafety
should be required for the address calculator at all.

This is also addressed by delaying reference creation. When an address
calculation does not result in reading of a value or in creation of a reference,
it is treated as safe and does not require `unsafe` scope.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, we introduce the concept of a raw reference (denoted here with `{raw T}`
and `{raw mut T}`. A raw reference is the type assigned to a `& place`
expression during type deduction when such an expression refers to a place
through means other than a direct reference (such as the target of a
dereference operation `&(*foo*)` or to fields of a union or a packed struct). 
Similar to integer literals, its type can not be named and only when the actual
type of a raw reference is needed it will coerce to reference or pointer but
its type will default to the *associated reference* type when unclear.

The current operators for address calculation from places (`*`, `&`, `&mut`,
`as &`, `as &mut`, `as *const` and `as *mut`, and `.field` expressions) also
interact with raw references. 

```
// Not UB, and safe.
let x = &packed.field as *const _;
        ^^^^^^^^^^^^^ is typed {raw T}

// UB and unsafe depending on `some_func`.
some_func(&packed.field);
         ^^^^^^^^^^^^^^^ coerced to reference or pointer

// unsafe without further type deduction information.
let y = &packed.field;
      ^ if no type information, coerced to reference.

// Safe and defined.
let z: *const _ = &packed.field;
                ^ coerced to pointer.
```

UB only happens when a raw reference is converted to a reference but the
referred to memory does not fulfill the reference invariants. An `unsafe` code
block is necessary when a raw reference is converted to reference or read from.
It is not necessary when the raw reference is cast to pointer or another raw
reference.  An explicit cast to a pointer can always be performed by an
explicit `as *const _`.

What this does not allow as defined behaviour or without `unsafe` blocks:

```
	vvvvvv required, cast from raw to reference
let x = unsafe { &packed.field as &F } as *const;
                               ^^^^^ explicit cast, likely UB during exec.

	vvvvvv required due to coercion to reference.
let x = unsafe { &packed.field };
     ^^^ probably UB, reference may not be aligned, initialized.
x.foo();
 ^ type deduction, x determined as reference type.
```

The above shows that any remaining `unsafe` block after this RFC should be
vetted with more care, even if it appears to only perform address calculation.
At the same time, it would be possible to perform multiple operations without
an explicit cast to/from pointer and dereferencing each time:

```
let x = loop {
    ^ is assigned a unique raw reference type,,,
    if parallel_signal() {
        break &packed.field;
	      ^^^^^^^^^^^^^ ... from this expression.
    }
};

let y: *mut _ = &x.subfield;
                ^^^^^^^^^^^ another raw reference expression.
```

On the level of MIR, a raw reference should be very similar to an actual
pointer. They are not assumed to have reference properties. Only at the point
of coercion or cast to reference will the reference semantics start.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Their behaviour for the sake of type deduction each expression will have a
unique type and the eventual type is determined by their usage only. Their
default type when under constrained is the corresponding reference type. These
values coerce implicitely. Allowed are coercion to *associated reference* with
same mutability and to *associated pointer* also with the same mutability.
Creation, and interaction and MIR level will be explained next.

For most parts, programmers need not know about raw reference to write code.
This RFC intends that there should be no way to actually denote this type in
type ascription.  Wherever this would be possible, the reference needs to be
cast to pointer or default to reference, the latter only being possible in
`unsafe` context. Since they work in a drop-in manner to actual references, the
difference will only matter when one needs to reason about the exact point
where undefined behaviour might happen. This point is delayed (hence the RFC
name) and thus a purely strong definedness guarantee.

```
// Will not compile in safe, two distinct types and no coercion to `& possible.
let x = if condition {
    &packed.field
} else {
    &packed.other_field
};

// But compiles in unsafe, coercion to `&`.
```


```
// Instead, you can write this:
let x = if  condition {
    &packed.field as *const_;
} else {
    &packed.other_field
     coerced to pointer ^^^
};
```

# Drawbacks
[drawbacks]: #drawbacks

This proposal complicates type deduction and reasoning. It is no longer clear
from code flow alone whether an expression will like `& (*place)` will be
interpreted as reference type, raw reference or eventually coerced to pointer.



Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is preferable over a separate dedicated syntax for creating
temporary raw references for some reasons:

* Old code that relied on `&packed.field as *const_` and incorrectly from lack
  of explicit other concepts inferred from this that any temporary reference
  creation eventually cast to pointer is defined should be considered. This
  reasoning would gain an explicit approval as long as the type is never
  explicitely cast to/ascribed with/used as a reference.

* It provides type-level reasoning powers and definitions that allows removal
  of `unsafe` from safe operations. The operation of dereferencing a pointer to
  a pointer to field does not actually dereference anything, it is mainly a
  notational convention inherited from similar languages such as C. This also
  makes other operations more intuitive, such as safely throwing away the
  result of `&packed.field` without UB.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

It should be clear what 

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
