# Generic Functions and Modules

I would like to reduce to the maximum the added syntax and semantics in our extensions.
After all, we are writing an extension, not a different language!
It should look and feel just like the original stuff and extensions should be straight forward, self-explanatory even.

Well, that would be the ideal case ofc.

## The proposal

It builds on 3 observations

* WGSL *already* has support for parametric types and functions, they are called [type-generators](https://www.w3.org/TR/WGSL/#type-generator) and [function overloads](https://www.w3.org/TR/WGSL/#builtin-functions). It even has a syntax for them.
  The issue is, they are not allowed to be declared in user code. This proposal changes this.
* A *module* is a collection of functions, structs and associated types that share a semantic. Turns out, a WGSL file is just that. This proposal introduces *implicit modules* as files.
* global value and alias declarations are immutable and must be initialized. What if we could override them? if so, this is a simple mechanism for module specialization.

## Type Hierarchy

Let's introduce the concept of type hierarchy. From the [conversion ranks](https://www.w3.org/TR/WGSL/#conversion-rank) we can deduce the type hierarchy for built-in types already:

* `u32` and `i32` are subtypes of `AbstractInt`, because `AbstractInt` can be converted to `u32` and `f32`
* `f32` and `f16` are subtypes of `AbstractFloat`, because `AbstractInt` can be converted to `u32` and `f32`
* `AbstractInt` is a subtype of `AbstractFloat`, because `AbstractInt` can be converted to] `AbstractFloat`
* `VecN<S>` is a subtype of `VecN<T>` if `S` is a subtype of `T`
* `MatCxR<S>` is a subtype of `MatCxR<T>` if `S` is a subtype of `T`
* `Array<S,N>` is a subtype of `Array<T,N>` if `S` is a subtype of `T`

And we can add to that list:
* `atomic<S>` is a subtype of `atomic<T>` if `S` is a subtype of `T`.
* `ptr<S>` is a subtype of `ptr<T>` if `S` is a subtype of `T`.

We can extend subtyping for user-defined types:
* structure type `S1` is a subtype of structure type `S2` if, for all members `m` in `S2`: `S1.m` exists and `S1.m` is a subtype of `S2.m`.
This allows extending structs by narrowing down a member type, or by adding new members.

And finally,
* `T` is a subtype of `T`
* `S` is a strict subtype of `T` if `S` is a subtype of `T` and `S` is not `T`

Those types are *leaf types* because they cannot have subtypes: `u32`, `i32`, `f32`, `f16`, `bool`. (or one alternative way to see it, would be that all literal values are also subtypes, then '10' is both an instance of `AbstractInt` and a subtype of it. But it doesn't matter here.)

## Type Constraints

First, we need to constrain a bit what can be a generic function parameter, because in the spec it is way too constrained. Consider this:

```rs
@const @must_use fn array<T, N>(e1 : T, ..., eN : T) -> array<T, N>
@must_use fn arrayLength(p: ptr<storage, array<E>, AM>) -> u32
```
This is the spec definition of built-in `array` constructor and function `arrayLength`.
`array` is parameterized by `T` and `N`. `arrayLength` is parameterized by `E` and `AM`.

* `T` and `E` must be one of: a `scalar`, `vector`, `matrix`, `atomic`, `array`, structure[^1].
* `N` must be an override-expression that evaluates to a `i32` or `u32`, and greater than 0[^2].
* `AM` must be a value of enumerant [`access mode`](https://www.w3.org/TR/WGSL/#access-mode)[^3].

So obviously, this level of constraining is too high to express in the grammar.

Instead, we could specify these constraints:
* Unconstrained: probably not very useful in real-world as we can do nothing with them, except zero-initialize.
* Constrained by: a user-defined struct, `vec`, `mat`, `array`, `atomic`, `ptr`, `AbstractInt` or `AbstractFloat`: Must be specialized by a constructible subtype.
* Constrained by a leaf type: `u32`, `i32`, `f32`, `f16`, `bool`: Must be specialized with a instance of that type (like the `N` parameter).

## Generic Functions

### Declaring generic functions

We can introduce a `@generic` attribute to define the generic parameters and constraints.
```rs
@generic(Num: AbstractFloat, T: MyStruct, N: u32)
fn my_fn(x: Num, t: T) -> array<Num>
```
The generic types can appear in the function formal parameter types, the return type and the function body.
They are not *required* to appear in the formal paramters.

TODO: Should they be allowed to be completely unused? Like Rust's PhantomData?

### Calling generic functions

Seems that explicit specialization is way simpler to implement.
Problem is, WGSL only has implicit specialization with the [overload resolution algorighm](https://www.w3.org/TR/WGSL/#overload-resolution-section).
We could also implement that in the future, but it's far from trivial.

explicit:
```
let res: MyStruct = my_fn<u32, MyStructExt, 10>(5, my_struct_inst);
```

## Generic Modules

TODO

## Remaining questions to answer

* Discord @mighdoll:
  > Is it true that vars are ok to duplicate? If a package has a var <workgroup> foo array<u32, 256>;, we wouldn't want two foos just because the package is used by two other packages.

* Do we want inline (explicit) modules too? if so, how do we resolve imports? -> this is the reason why Rust has the `mod` declaration afaik. In Rust, the tree of modules is built somewhat independently from the filesystem.

## Possible future extensions

* variadic function parameters are supported by the built-in functions, e.g. [the array constructor](https://www.w3.org/TR/WGSL/#array-builtin). We could support them too.
* the `@const` and `@must_use` function attributes are not allowed in user code either. We could support them too.
* Accepting `AbstractInt`/`AbstractFloat` and returning them from functions? It could make sense in some cases.



[^1]: https://www.w3.org/TR/WGSL/#array-types
[^2]: https://www.w3.org/TR/WGSL/#array-builtin
[^3]: https://www.w3.org/TR/WGSL/#arrayLength-builtin
