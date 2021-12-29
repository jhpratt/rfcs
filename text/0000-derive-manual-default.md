- Feature Name: `derive_manual_default`
- Start Date: 2021-12-29
- RFC PR: TODO
- Rust Issue: TODO

# Summary
[summary]: #summary

Users can provide default values for individual fields when deriving `Default` on `struct`s, thus
avoiding the need to write a manual implementation of `Default` when the type's default is
insufficient.

```rust
#[derive(Default)]
struct Window {
    width: u16 = 640,
    height: u16 = 480,
}

assert_eq!(Window::default(), Window { width: 640, height: 480 });
```

# Motivation
[motivation]: #motivation

## Boilerplate reduction

The `#[derive(..)]` ("custom derive") mechanism works by defining procedural macros. Because they
are macros, they operate on abstract syntax and don't have more information available. Therefore,
when you `#[derive(Default)]` on a data type definition such as:

```rust
#[derive(Default)]
struct Foo {
    bar: u8,
    baz: String,
}
```

it only has the immediate "textual" definition available to it.

Because Rust currently does not have an in-language way to define default values, you cannot
`#[derive(Default)]` in the cases where you are not happy with the default values that each field's
type provides. By extending the syntax of Rust such that default values can be provided,
`#[derive(Default)]` can be used in many more circumstances and thus boilerplate is further reduced.

## Usage by other `#[derive(..)]` macros

[`serde`]: https://serde.rs/attributes.html

Custom derive macros exist that have a notion of or use default values.

### `serde`

For example, the [`serde`] crate provides a `#[serde(default)]` attribute that can be used on
`struct`s and fields. This will use the field's or type's `Default` implementations. This works well
with field defaults: `serde` can either continue to rely on `Default` implementations, in which case
this RFC facilitates specification of field defaults, or it can directly use the default values
provided in the type definition.

### `structopt`

Another example is the `structopt` crate with which you can write:

```rust
#[derive(StructOpt)]
#[structopt(name = "example", about = "An example of StructOpt usage.")]
struct Opt {
    #[structopt(short = "s", long = "speed", default_value = "42")]
    speed: f64,
}
```

By having default field values in the language, `structopt` could let you write:

```rust
#[derive(StructOpt)]
#[structopt(name = "example", about = "An example of StructOpt usage.")]
struct Opt {
    #[structopt(short = "s", long = "speed")]
    speed: f64 = 42,
}
```

### `derive_builder`

[`derive_builder`]: https://docs.rs/derive_builder/0.7.0/derive_builder/#default-values

A third example comes from the crate [`derive_builder`]. As the name implies, you can use it to
`#[derive(Builder)]` for your types. An example is:

```rust
#[derive(Builder)]
struct Lorem {
    #[builder(default = "42")]
    pub ipsum: u32,
}
```

This can similarly be simplified to:

```rust
#[derive(Builder)]
struct Lorem {
    pub ipsum: u32 = 42,
}
```

### Conclusion

As seen in the previous sections, rather than make deriving `Default` more magical, by allowing
default field values in the language, user-space custom derive macros can make use of them.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Deriving `Default`

Previously, you might have instead implemented the `Default` trait like so:

```rust
impl Default for Probability {
    fn default() -> Self {
        Self(0.5)
    }
}
```

However, since you can specify `f32 = 0.5` in the definition of `Probability`, you can take
advantage of that to write the simpler and more idiomatic:

```rust
#[derive(Default)]
pub struct Probability(f32 = 0.5);
```

Having done this, a `Default` implementation equivalent to the former will be generated for you.

## Default fields values are [`const` context]s
[`const` context]: https://github.com/rust-lang/reference/blob/06f9e61931bcf58b91dfe6c924057e42ce273ee1/src/const_eval.md

When you provide a default value `field: Type = value`, the given `value` must be a *constant
expression* such that it is valid in a [`const` context]. Therefore, you can **not** write something
like:

```rust
fn launch_missiles() -> Result<(), LaunchFailure> {
    authenticate()?;
    begin_launch_sequence()?;
    ignite()?;
    Ok(())
}

struct BadFoo {
    bad_field: u8 = {
        launch_missiles().unwrap();
        42
    },
}
```

Since launching missiles interacts with the real world and has *side effects* in it, it is not
possible to do that in a `const` context since it may violate deterministic compilation.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Grammar

The following grammars shall be extended to support default field values:

```text
TupleField = attrs:OuterAttr* vis:Vis? ty:Type { "=" def:Expr }?;
RecordField = attrs:OuterAttr* vis:Vis? name:IDENT ":" ty:Type { "=" def:Expr }?;
```

## Defining defaults

Given a field where the default is specified, i.e. either:

```text
RecordField = attrs:OuterAttr* vis:Vis? name:IDENT ":" ty:Type "=" def:Expr;
TupleField = attrs:OuterAttr* vis:Vis? ty:Type "=" def:Expr;
```

both of the following rules apply when type-checking:

1. The expression `def` must be a constant expression.
2. The expression `def` must unify with the type `ty`.

When lint attributes such as `#[allow(lint_name)]` are placed on a field, they also apply to `def`
if it exists.

## `#[derive(Default)]`

When generating an implementation of `Default`, the compiler shall emit an expression where the
user-provided default value is used to initialize the field, rather than the default value of the
field's type. For example,

```rust
#[derive(Default)]
struct Window {
    width: u16 = 640,
    height: u16 = 480,
}
```

shall generate

```rust
impl ::core::default::Default for Window {
    fn default() -> Self {
        Self {
            width: 640,
            height: 480,
        }
    }
}
```

All fields where the default is not specified will remain initialized to their type's default
value.

### Bounds

The bounds of the generated `Default` implementation are not affected by this RFC.

# Drawbacks
[drawbacks]: #drawbacks

- This integrates the concept of a default value into the language itself, rather than solely
  existing in the standard library. Note that this does _not_ necessitate making the `Default` trait
  a lang item.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
[RFC 3107]: https://rust-lang.github.io/rfcs/3107-derive-enum-default.html

This proposed syntax is, to an extent, expected: there is currently a diagnostic in place
indicating that this syntax isn't supported.
One alternative is to extend the `#[default]` attribute introduced in [RFC 3107] to accept default
values. The drawback of this is that the interaction with a future possibility from RFC 3107 is
not clear:

```rust
#[derive(Default)]
enum Foo {
    #[default = …] // What should this be? `Foo::Bar { .. }`? Just `Bar { .. }`? Neither?
    Bar {
        a: u8,
        b: u8,
    },
    Baz {
        a: u8,
        b: u8,
    },
}
```

## Provided associated items as precedent

While Rust does not have any support for default values for fields or for runtime parameters of
functions, the notion of defaults is not foreign to Rust as a language feature.

Indeed, it is possible to provide default function bodies for `fn` items in `trait` definitions. For
example:

```rust
pub trait PartialEq<Rhs: ?Sized = Self> {
    fn eq(&self, other: &Rhs) -> bool;

    fn ne(&self, other: &Rhs) -> bool { // A default body.
        !self.eq(other)
    }
}
```

In traits, `const` items can also be assigned a default value. For example:

```rust
trait Foo {
    const BAR: usize = 42; // A default value.
}
```

Thus, to extend Rust with a notion of field defaults is not an entirely alien concept.

# Prior art
[prior-art]: #prior-art

???

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- None so far.

# Future possibilities
[future-possibilities]: #future-possibilities

- The derived `Default` implementation could be made `const` so long as types without a specified
  default value `impl const Default`.
- Support could be extended to the `#[default]` variant of `enum`s once non-unit variants are
  supported.
