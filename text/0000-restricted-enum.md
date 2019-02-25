- Feature Name: (restricted_variants)
- Start Date: (2019-02-23)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add trait bound clauses to enum variants, allowing several additional concise
code patterns and aligning the structurally inferred marker traits with the sum
semantics of enums.

# Motivation
[motivation]: #motivation

This is perhaps best motivated by an example:

```
pub enum Synced<T: Send> {
    /// The contained value is synced with a mutex.
    MaybeUnsync(Mutex<T>),

    /// The value is already synced on its own.
    Sync(T) where T: Sync,
}

fn assert_sync<T: Sync>() { }

fn always_sync<T: Send>() {
    /// Currently impossible, would need `T: Sync`
    assert_sync::<Synced<T>>()
}
```

The `Synced` type makes it possible to create a `Sync` capsule around any type
but avoids the overhead of `Mutex<_>` where it is not necessary. Which of these
were chosen is revealed at the site of a match on this enum.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Automatic traits and the construction will now consider each enum variant on
its own. Without a trait bound clause, this is equivalent to the current
system. But an additional guard clause allows restricting usage of a single
variant without affecting the generic argument bounds of the enum type itself.
The extra restrictions are then provided back to the programmer through
`match`, by allowing access to the gated `impl` when matching that variant.

A maybe common occurance is an I/O algorithm that can be implemented more
efficiently with `BufRead` but for which the interface should not depend on it.
This kind of function is not adequately captured by current specialization and
would require an additional trait and two impls to dispatch.  Using `enum` for
this makes everything much clearer and declares the possibilities in a single
location that can be freely chosen to be part of SemVer or not. Additionally,
the caller may choose not to use the specialized version (e.g. if they
determine that some heuristic makes the generic impl more efficient than the
specialized version for some kind of reader).

```
pub enum MaybeBuf<T: Read> {
    Standard(T),
    Buf(T) where T: BufRead,
}

pub fn do_io<T: Read>(read: MaybeBuf<T>) {
    match read {
        MaybeBuf::Standard(reader) => dispatch_standard(reader),
	MaybeBuf::Buf(reader) => dispatch_buf(reader),
    }
}

fn dispatch_standard<T: Read>(reader: T) { ... }
fn dispatch_buf<T: BufRead>(reader: T) { ... }
```

Another common case is encapsulating a related pair of values while permitting
a size optimization in case the representation for both is already present in a
single type. This could otherwise be solved with a specialized trait and an
automatic impl for generic `T` but that route may run into conflicting impl
blocks, especially if the trait should be implementable by a third-party
library as well. Using a constrained enum variant to disambiguiate would not
require a generic and broad impl block and be much more concise:

```
pub trait Value {
    fn get(&self) -> usize;
}

pub trait Named {
    fn name(&self) -> &str;
}

pub enum KeyValue<T: Value> {
    Pair(String, T),
    SelfNamed(T) where T: Named,
}

impl<T: Value> KeyValue<T> {
    fn name(&self) -> &str {
        match self {
	    KeyValue::Pair(name, _) => name,
	    KeyValue::SelfNamed(val) => val.name(),
	}
    }
}

//! Which would otherwise have to look like:
pub strut Kv<T>(pub T);

pub trait KeyValue: Value {
    fn name(&self) -> &str;
}

impl<T: Value + Named> KeyValue for Kv<T> {
    fn name(&self) -> &str {
    	self.0.name()
    }
}

/// Uh oh, dangerously close to conflicting impl.
///
/// tuple (`String`, T) better not be `Value + Named`.
impl<T: Value> KeyValue for Kv<(String, T)> {
    fn name(&self) -> &str {
    	&self.0
    }
}
```

Now that we've seen the use, let us quickly state the straightforward
restriction on construction. The implicit functions provided by enum tuple
variants are simple extended to also include their own trait bounds. The
compiler error message should hint at which bound is violated through something
like:

```
pub enum MaybeBuf<T: Read> {
    Standard(T),
    Buf(T) where T: BufRead,
}

fn main() {
    // Note: stdin() is not buffered like StdinLock.
    MaybeBuf::Buf(stdin());
}

error[E0000]: the trait bound `std::io::Stdin: std::io::BufRead` is not satisfied
 --> src/main.rs:8:8
  |
8 |     MaybeBuf::Buf(stdin());
  |     ^^^^^^^^^^^^^^^^^^^^^^ the trait `std::io::BufRead` is not implemented for `std::io::Stdin`
  |
note: required by enum variant `MaybeBuf::Buf` 
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The goal is to elevates the power of enums beyond tag dispatch, improving
structural induction for marker traits while also allowing more general type
representations. Trait bounds are allowed on each enum variant where they are
asserted when constructed and provided to a match expression. Marker traits are
inferred by inspecting each variant individually with its bounds before
performing the structural induction otherwise.

```
/// Permit passing a value by reference, and a sized value by value.
///
/// Note that this type is always `Sized` as each variant is `Sized`.
enum ValOrRef<'a, T: ?Sized + 'a> {
    /// `&_` is always sized.
    Ref(&'a T),

    /// This is sized due to the bound.
    Val(T) where T: Sized,
}

impl<'a, T: ?Sized + 'a> ValOrRef<'a, T> {
    /// This requires `Self: Sized`.
    fn get(self) -> T where T: Clone {
        match self {
	    ValOrRef::Ref(t) => t.clone(),
	    ValOrRef::Val(t) => t,
	}
    }
}
```

An enum variant provides, in a way, a witness for the additional trait bounds
mentioned in its variant clause. This way, the generic interface of an enum can
offer a less restricted interface with individual variants providing more
concrete trait bounds. This provides possibilities similar to runtime
specialization. 

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

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
