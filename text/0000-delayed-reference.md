- Feature Name: delayed_reference
- Start Date: 2019-01-20
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Introduce an additional unnamed type and a corresponding conversion operation
to references that safely explain the desired behaviour of 'rereference-as-ptr'
expressions such as `&(*foo).field as *const _`. Also, the resulting MIR will
not exhibit undefined behaviour. That is, it retroactively makes typical
statements defined even for packed structs and requires less unsafe. The syntax
is intended to be backwards compatible in a non-breaking manner.

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
that tries to approach this problem by adding a MIR operation that performs a
direct pointer to pointer-to-field conversion in a few defined syntactical
cases.  This has the immediate drawback of leaving some existing code at a
state of undefined behaviour and introducing UB interact with no-op pointer
casts:

```
// UB, intermediate & is not allowed.
&packed.field as &T as *const T;

// UB, throwing away the result is not sound.
&packed.field;
```

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

The result of the some borrow expressions is changed to a *raw reference*.
Specifically, when such a place expression of the form `(*ptr).field` where
`ptr` is a pointer or a `raw reference`. The type of a raw reference can not be
named, and it can be coerced to a standard reference or a pointer according to
context. It is denoted here with `{raw T}` and `{raw mut T}` but every
expression has a unique type, similar to closures. It will be safe to
dereference a pointer for this case, and this case only, while it will still
remain unsafe to coerce the raw reference to a normal reference. The reference
to which a raw reference can coerce is unique and we will call it associated
reference in this document.

For most parts, programmers need not know about raw reference to write code.
This RFC intends that there should be no way to actually denote this type in
type ascription.  Wherever this would be possible, the reference needs to be
cast to pointer or default to reference, the latter only being possible in
`unsafe` context. Since they work in a drop-in manner to actual references, the
difference will only matter when one needs to reason about the exact point
where undefined behaviour might happen. This point is delayed compared to
immediately interpreting borrow expressions as references (hence the RFC name)
and thus yields a purely stronger definedness guarantee.

The current operators for interact with raw references in the same manner as
with standard pointers (`*`, `as &`, `as &mut`, `as *const` and `as *mut`, and
`.field` expressions). This includes that `& (*raw_ref).field` would yield
another raw reference.

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
of coercion or cast to reference will the reference semantics start. This could
potentially reuse only existing MIR operators by changing their semantics but
instead it is proposed to explicitly assign the new expressions a type 
`&raw _`. If the conversion operation should also be a new instruction is
mostly a matter of taste, we could also overload the semantics of `as &T` to
also cast raw references.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Reference discussion is split in four parts: Firstly, the proposed type nature
is explained, followed by examples and diagnostics. Then, the backwards
compatibility is examinated and lastly the MIR level changes are listed, which
will turn out to be more or less decoupled from the surface syntax.

For the sake of type deduction each expression will have a unique type and the
eventual type is determined by their usage only. Their default type when under
constrained is the corresponding reference type. These values coerce
implicitely. Allowed are coercion to *associated reference* with same
mutability and to *associated pointer* also with the same mutability.
Creation, and interaction and MIR level will be explained next.

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

Let's talk about error messages, since after this proposal most address
calculation should be written in safe context. Thus, a good error message
should be emitted when a program contains a raw reference coercion to reference
in safe code. This message should allude to explicit requirement of unsafe for
casting to reference and should note the chain of raw reference types leading
to this error. Furthermore, there should be different messages when the
compiler inferred the reference type by ambiguity.

Suppose we forget an `unsafe`, but it is clear that reference type is our only
option for type deduction. Take the earlier example:

```
// Fails to compile!!
let x =   &packed.field;
        ^ not in an unsafe block.
x.foo();
 ^ type deduction, x determined as reference type.
```

```
error[E0000]: unsafe reference to packed field
  |
1 |  let x = &packed.field;
  |        ^ cast to reference requires unsafe { } block

Note: This expression refers to the field of a packed struct. Safe code can
      only point to such fields but it can not create a reference.
  |
1 |  let x = &packed.field;
  |          ^^^^^^^^^^^^^

Note: reference type inferred here:
  |
2 | x.foo();
  |

```

Compare this to the branch case where reference type is inferred to resolve the
ambiguity of two different raw reference expressions. Note that this will *not*
produce an error when it occurs in an unsafe code block.

```
// Fails to compile!!
let x = if condition {
    &packed.field
} else {
    &packed.other_field
};
```

```
error[E9999]: implicit unsafe cast to reference type.
1 |  let x = if condition {
2 |      &packed.field
  |      ^^^^^^^^^^^^^ this value is implicitly converted to reference
3 |  } else {

Note: The reference type was inferred as the type of this expression because no
      expression supplying the result has a concrete type.
1 | let x = if condition {
...         ^^^^^^^^^^^^^^
3 | } else {
... ^^^^^^^^
5 | };
  | ^^

Note: Maybe you meant to explicitly cast one of them to a pointer?
1 |  let x = if condition {
2 |      &packed.field as *const_
  |                    ^^^^^^^^^^^
3 |  } else {
```

Third, I will get into a few more details about the backwards compatibilty of
this proposal. There is one additional rule for the new type: An unsafe block
will always coerce its result into a normal reference.

The new expression type can only be coerced to a pointer when this has been
asserted through direct type ascription. In these cases, a reference would also
coerce to a pointer. Whenever the compiler has to consider unification of two
raw types, there are three new cases:

* One is a raw reference while the other is pointer type. In this case, we
  coerce to pointer just as the reference before would have.
* One is a raw reference while the other is a reference type. In this case, we
  coerce to a reference *only because we are in an unsafe block*. Outside of an
  unsafe block, this will fail but this is not a problem since this case can
  not happen for legacy code. Any raw reference in legacy code are created
  within unsafe blocks, and coerced to normal reference when leaving them.
* This is very similar, we also coerce both to a reference.


Finally, on the MIR level we make three changes:

1. Introduce a new operator `&[const|mut] raw <place>` to create a new type, a
   `&[const|mut] raw` reference.
2. Change the compilation of taking a reference to a place through means other
   than an existing reference to instead emit this new instruction.
3. Convert the new type to one of `&[mut]` or `*[const|mut]` when it is coerced
   to one of these in the surface language.

The last change is subject to a decision of simply reusing `as *const` and 
`&(*place)` operations or introducing a new operator. While the former require
less changes to MIR, the latter offers the possibility of being an injection
point for Rust `UBsan`. I argue, that the `&(*)` operator should in MIR only be
applied to references in the first place.

Since it is at these instructions were references are created out of thin air
by assertion of the programmer, it would be a primary opportunity to insert
additional code checking these assertions.


# Drawbacks
[drawbacks]: #drawbacks

This proposal complicates type deduction and reasoning. It is no longer clear
from code flow alone whether an expression will like `& (*place)` will be
interpreted as reference type, raw reference or eventually coerced to pointer.

This increases the number of type considerations in surface level code and MIR.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is preferable over a dedicated blessed syntax for creating
temporary raw references for some reasons:

Old code that relied on `&packed.field as *const_` and incorrectly from lack of
explicit other concepts inferred from this that any temporary reference
creation eventually cast to pointer is defined should be considered. This
reasoning would gain an explicit approval as long as the type is never
explicitly cast to/ascribed with/used as a reference.

It provides type-level reasoning powers and definitions that allows removal of
`unsafe` from safe operations. The operation of dereferencing a pointer to a
pointer to field does not actually dereference anything, it is mainly a
notational convention inherited from similar languages such as C. This also
makes other operations more intuitive, such as safely throwing away the result
of `&packed.field` without UB.

The operation of creating a reference from a non-reference place is always
explicit in MIR, as the current instructions are separated into two: One
dedicated to raw address calculation and one for asserting the reference
properties. There is no longer an overlap with the unconditionally safe
operation for constructing a reference to a field of place pointed to by
another reference.


# Prior art
[prior-art]: #prior-art

An [alternate proposal](https://github.com/rust-lang/rfcs/pull/2582) addresses
the same underlying concern als with MIR changes. These are not exclusive and
MIR-only changes are generally preferable to surface language changes. This
proposal tries to make these changes more consistent at both levels. 

The split in MIR instructions, both the new type and its creation, are
compatible with the mechanisms outlined in that other rfc. We can, mostly, use
both MIR changes to express the fundamental operation of getting a point to a
place without going through a reference. The other text is a less intrusive fix
to the immediate problem of having **no** semantically defined way of achieving
the require effect. In comparison, the changes here are more fundamental but
also more principled. We try to avoid `&(*)` in any context where we are not
positive on the reference nature of our target. The additional guarantees
asserted by unsafe code are instead expressed via a separate instruction.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The MIR instruction design is largely independent of the surface level type
changes. In theory, one could replace MIR before implementing the surface level
changes or one could make the surface level defined by using other MIR
instructions that do not rely on references to create pointers.

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

With arbitrary self-types it could become possible to safely call methods on
the pointer of a place expression in a single statement with the same safe
syntax. as coercion converts these to pointer type.

```
impl Foo {
    fn init(self: *mut Self) { }
}

union Foobar {
    _uninit: (),
    some_struct: Foo,
}

let _ = foobar.some_struct.init();
                         ^^^ methods on a raw reference?
```

Note that this is *not* proposed here as the method does not name a field.
It remains open if constructs such as these could also performed safely by
omitting the intermediate reference coercion, the semantics still remain
similar to this:

```
let _t: &_ = &foobar.some_struct;
        ^^ unsafe coercion to reference here.
_t.init();
 ^^^ No such method on type `Foo`
```

