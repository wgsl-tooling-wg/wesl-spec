# Generic Declarations and Generic Modules

I would like to limit as much as possible the amount of additional syntax and semantics in our extensions.
After all, we are writing an extension, not a different language!
It should look and feel just like the original stuff and extensions should be straight forward, self-explanatory even.

## The proposal

It builds on 3 observations

* WGSL *already* has support for parametric types and functions, they are called [type-generators](https://www.w3.org/TR/WGSL/#type-generator) and [function overloads](https://www.w3.org/TR/WGSL/#builtin-functions). It even has some syntax for them.
  The issue is, they are not allowed to be declared in user code. This proposal changes this.
* A *module* is a collection of functions, structs and associated types that share a semantic. Turns out, a WGSL file is just that. This proposal introduces *implicit modules* as files.
* global value and alias declarations are immutable and must be initialized. What if we could override them? if so, this is a simple mechanism for module specialization.
  What if we also could override struct and functions declarations? In fact, all declarations?

It proposes two extensions: [**Generic Declarations**](#generic-declarations) and [**Generic Modules**](#generic-modules) (+ optional [**Function-as-Type**](#function-as-type) [**Module-as-Type**](#module-as-type)).
They are orthogonal but they share the concepts of [*Type Hierarchy*](formalism-type-hierarchy) and [*Generic Type*](#formalism-generic-type) formalized below.
We could support only one of the two and achieve the same expressivity AFAIK. But not the same ergonomics.

## Formalism: Type Hierarchy

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
* structure type `S1` is a subtype of structure type `S2` if, for all members `m` in `S2`: `S1.m` exists and `S1.m` is a subtype of `S2.m` (member covariance).
This allows extending structs by narrowing down a member type, or by adding new members.
* (optional) function signature `F1` is a subtype of function signature `F2` if,
  * `F1` and `F2` have the same number of formal parameters,
  * the `i`th formal parameter of `F1` is a subtype of the `i`th formal parameter of `F2` (parameter covariance),
  * the `F1`'s return type is a subtype of `F2`'s return type (return type covariance).

And finally some conventions:
* `T` is a subtype of `T`
* `S` is a strict subtype of `T` if `S` is a subtype of `T` and `S` is not `T`
* We note `S: T` to indicate that `S` is a subtype of `T`. This applies transitively.

Those types are *leaf types* because they cannot have subtypes: `u32`, `i32`, `f32`, `f16`, `bool`.
> (or one alternative way to see it, would be that all literal values are also subtypes, then '10' is both an instance of `AbstractInt` and a subtype of it. But it doesn't matter here.)

## Formalism: Generic Type

A generic type is associated with a type constraint. It can be replaced by any other type that is a subtype of the constraint.

Said formally: Let `T` be a generic type and `C` its constraint (`T: C`). Then `U` can *specialize* `T` if `U: C`.

A [generic declaration](#generic-declarations) (or built-in *type generator*) can be specialized with a generic type. That specialized type is also a generic type, e.g. `array<T>` is a generic type.

### Operations on generic types

In generic code, the following operations are possible:
* instantiating a `T` with the zero-valued constructor,
* calling a generic function by specializing its generic parameter `U: D` with `T` if `T: D`,
* if `C` is a struct type, accessing members of `C` from an instance of `T`,
* if `C` is an array or vector type, using the array indexing operator on an instance of `T`,
* if `C` is a ptr type, using the dereference operator on an instance of `T`,
* if `C` is a number type, using any built-in operations on numbers that are implemented for all subtypes of `C`,
* ...etc.

But the following is NOT possible:
* assigning a `C` to a `T` (since `T` is a subtype of `C`, not necessarily equal)
* calling a function that takes a parameter of type `C` with an instance of `T` (since `T` is a subtype of `C`, not necessarily equal)
* ...etc.

### Constraining generic types

Note that compared to the WGSL spec, we need to constrain a bit more what can be a generic type. Consider this example from the spec:

```rs
@const @must_use fn array<T, N>(e1 : T, ..., eN : T) -> array<T, N>
@must_use fn arrayLength(p: ptr<storage, array<E>, AM>) -> u32
```
`array` is parameterized by `T` and `N`. `arrayLength` is parameterized by `E` and `AM`.
* `T` and `E` must be one of: a `scalar`, `vector`, `matrix`, `atomic`, `array`, structure[^1].
* `N` must be an override-expression that evaluates to a `i32` or `u32`, and greater than 0[^2].
* `AM` must be a value of enumerant [`access mode`](https://www.w3.org/TR/WGSL/#access-mode)[^3].

So obviously, this level of constraining is too high to express in the grammar.

TL;DR instead, we could specify these constraints:
* Unconstrained (or implicitly constrained by [AllTypes](https://www.w3.org/TR/WGSL/#alltypes-type).
* Constrained by: a user-defined struct, `vec`, `mat`, `array`, `atomic`, `ptr`, `AbstractInt` or `AbstractFloat`: Must be specialized by a *constructible* subtype.
* Constrained by a leaf type: `u32`, `i32`, `f32`, `f16`, `bool`: Must be specialized with a instance of that type (like the `N` parameter).

## Generic Declarations

We introduce a `@generic` attribute to define the generic parameters and constraints.
```rs
@generic(Num: AbstractFloat, T: MyStruct, N: u32)
fn my_fn(x: Num, t: T) -> array<Num, N>

@generic(T)
struct Point2D {
  x: T,
  y: T,
}

@generic(T: AbstractInt, W: u32, H: u32)
alias matrix = array<array<T, H>, W>;
```

### Using generic declarations

I chose the template list notation, as it seems to be unambiguous in the grammar and is a familiar notation for generics. Besides, it is already used for type generators.

```rs
let res: array<f32, 10> = my_fn<f32, SomeStruct, 10>(3.5f, struct_inst);

@generic(T: AbstractFloat)
fn rotate(point: Point2D<T>, angle: f32) -> Point2D<T> {}

alias mat4x4f = matrix<f32, 4, 4>;
```

## (optional) Function-as-Type

Since a function signature now has a well-defined type and subtyping, we *could* support "function pointers" via genericity.

You cannot pass by value a function pointer, instead it is passed as a specialization of the generic parameter.

This can be implemented simply by substitution of the generic type, like the rest of the generics. No extra work needed except for the type-checking.

```rs
@generic(T)
fn callback_signature(arg: T) -> f32; // just define an abstract function, this serves only the purpose of expliciting the signature.

@generic(T, F: callback_signature<T>)
fn my_fn() -> f32 {
  // ...
  return F(val2); // call the generic function (similar to a function pointer).
}

fn callback_implementor(arg: u32) -> f32 { // this function signature is a subtype of callback_signature
  // ...
}

fn main() {
  my_fn<u32, callback_implementor>();
}
```

See [the map reduce example](#map-reduce).

## Generic Modules

With a global declaration override mechanism we can implement specialization of modules. We *could* specialize any global declaration:
* constants (most straightforward),
* type aliases (implements generic associated types),
* functions (implements function overriding),
* structs (implements genericity over data structures).

Overriding declarations could be done at the import site, or by via the linker configuration.

An override *must* be a subtype of the overridden.

### (optional): uninitialized declarations

A straightforward extension would be to allow uninitialized declarations.
Module that have uninitialized declarations are *generic*, and *must* be specialized at import-time to be used.

* uninitialized consts:
  ```rs
  const texture_bind_group: u32;
  ```
* uninitialized type aliases (can implement Generic Associated Types):
  ```rs
  alias T;
  // or perhaps, with an optional type constraint
  alias T: AbstractFloat;
  ```
* uninitialized functions (can implement interfaces):
  ```rs
  // in a module shape.rs
  struct ShapeBase { ... }
  alias Shape: ShapeBase;
  fn compute_volume(shape: Shape) -> f32;
  ```
* uninitialized structs? I don't see a use-case but why not.

### Overriding declarations at import-time

> I know that we are not settled on the import syntax and semantics, so I just give an overview here.

We introduce a `@override` attribute to substitute declarations in modules. Both uninitialized and initialized declarations can be overridden.
```
@override(texture_bind_group: 3)
import util/sample_texture/{sample};
```

The same module can be imported multiple times with different specializations. This is equivalent to importing from two different files.
```rs
@override(texture_bind_group: 3, texture_type: u32)
import util/sample_texture/{sample as sample_u32};

@override(texture_bind_group: 5, texture_type: f32)
import util/sample_texture/{sample as sample_f32};
```

Importing the whole module is possible with the star (`*`) import.

> NOTE: the declaration you provide as an override cannot access any of the declarations in the imported module. But it can access all declarations in the module where it is defined (see [examples](#examples)).

### Overriding declarations via the linker

This should be implementation-defined, but basically
```ts
const linker = new Linker();
const linkerOptions = {
  overrides: {
    "some/file.wesl": { "texture_type": "u32" },
    "some_library/some/file.wesl": { "group_offset": 10 },
  }
};
let wgsl_source = linker.link("main.wesl", linkerOptions)
```

## (optional) Module-as-Type

We can allow a module to be passed as a type constraint. Then, a declaration with a module generic can be specialized with any other module that overrides all uninitialized declarations. This can help implement object-oriented or interface semantics.

Formally, a module `M1` is a subtype of module `M2` if,
* for all declarations `D1` in `M1` that override a declaration `D2` in `M2`, `D1` is a subtype of `D2`.
* all uninitialized declarations in `M2` are overridden in `M1`.

For this, a declaration can be decorated with the `@override()` attribute that specifies which declaration of which module it overrides.
```rs
// file animal.wesl
fn say_hi();
fn walk();

// file cat.wesl
import animal/*;
@override(animal::say_hi)
fn meow() { ... }
@override(animal::walk)
fn walk() { ... }
```

```rs
import animal as animal_mod;
import cat;

@generic(A: animal_mod)
fn simulate_animal() {
  A::say_hi();
  A::walk();
  ...
}

fn main() {
  simulate_animal<cat>(); // calls cat::meow() and cat::walk()
}
```

## Implementation

Type checking does not have to be enforced by the linker, especially for runtime linking where we care about performance.
All of this proposal (except module-as-types) can be implemented with simple substitution.
A compile-time linker or a IDE linter/langserver should implement type checking. Basically what Typescript does. 

## Remaining questions to answer

* Discord @mighdoll:
  > Is it true that vars are ok to duplicate? If a package has a `var <workgroup> foo array<u32, 256>;`, we wouldn't want two foos just because the package is used by two other packages.

* Do we want inline (explicit) modules too? if so, how do we resolve imports? -> this is the reason why Rust has the `mod` declaration AFAIK. In Rust, the tree of modules is built somewhat independently from the filesystem.

## Possible future extensions

* variadic function parameters are supported by the built-in functions, e.g. [the array constructor](https://www.w3.org/TR/WGSL/#array-builtin). We could support them too.
* the `@const` and `@must_use` function attributes are not allowed in user code either. We could support them too.
* Accepting `AbstractInt`/`AbstractFloat` and returning them from functions? It could make sense in some cases.
* Should we also allow using the generic parameters explicitly for built-in functions, for coherence?
* Do we want to forbid overriding certain declarations? Perhaps a `@final` attribute?

## Examples

### Object Oriented

This example uses function generics, module generics and module-as-type.

```rs
// file shape.rs: it acts as an interface (or a base class in OOP).
struct Shape { color: vec4f }
fn compute_area(shape: Shape) -> f32;
fn is_inside(shape: Shape, point: vec2f) -> bool;
fn rotate(shape: Shape, angle: f32) -> Shape;
fn translate(shape: Shape, translation: vec2f) -> Shape;
fn color_at(shape: Shape, point: vec2f) -> vec4f { return shape.color; } // default implementation, but can be overriden e.g. is a shape has a texture

// file shape/square.rs
import ../shape/*;
@override(shape::Shape)
struct Square { color: vec4f, pos: vec2f, size: vec2f }
@override(shape::compute_area)
fn compute_area(square: Square) -> f32 { return square.size.x * square.size.y; }
@override(shape::is_inside)
fn is_inside(shape: Shape, point: vec2f) -> bool { return point.x > shape.pos.x && ... }
@override(shape::rotate)
fn rotate(square: Square) -> Square { ... }
@override(shape::translate)
fn translate(square: Square, translation: vec2f) -> Square { ... }

// file main.rs
import shape/*; // module import (the module is generic so some of its functions cannot be used)
import shape/square/{*, Square}; // module import and struct import (the module is concrete)

@group(0) @binding(0)
const time: f32;

@generic(S: shape) // generic over a module, so this function can be used with any module that is a subtype of (aka. implements) shape.
fn draw_shape(my_shape: shape<S>::Shape, translation: vec2f, rotation: f32, pixel_coord: vec2f) -> vec4f {
  let my_shape = S::translate(my_shape, translation);
  let my_shape = S::rotate(my_shape, rotation);
  if S::is_inside(my_shape, pixel_coord) {
    return S::color_at(pixel_coord);
  } else {
    return vec4f(0.0); // background color
  }
}

@fragment
fn frag_main(@builtin(position) clip_pos) -> vec4f {
  let pixel_coord = (clip_pos + 0.5) * 100.0; // screen is 100x100 pixels
  let my_square = Square { pos: vec2f(10.0, 20.0), size: vec2f(30.0, 40.0) };
  let pixel_color = draw_shape<square>(my_square, vec2f(5.0, 5.0), time, pixel_coord);
  return pixel_color;
}
```

### Map Reduce

This example uses function generics and function-as-type

```rs
@generic(T)
fn map_fn(x: T) -> T; // just here to provide the type signature, doesn't have to be implemented.

@generic(T, N: u32, F: map_fn<T>)
fn array_map(arr: array<T, N>) -> array<T, N> {
  for (var i = 0u; i < N; i++) {
    arr[i] = F(arr[i]);
  }
  return arr;
}

@generic(T, U)
fn reduce_fn(acc: U, x: T) -> U; // just here to provide the type signature, doesn't have to be implemented.

@generic(T, U, N: u32, F: reduce_fn<T, U>)
fn array_reduce(init: U, arr: array<T, N>) -> U {
  var acc = init;
  for (var i = 0u; i < N; i++) {
    acc = F(acc, arr[i]);
  }
  return acc
}

@generic(T: AbstractFloat)
fn times_two(val: T) -> T { return val * 2.0; }

@generic(T: AbstractFloat)
fn sum(a: T, b: T) -> T { return a + b; }

fn main() {
  let arr = array(1.0, 2.0, 3.0, 4.0);
  let mapped = array_map<f32, 4u, times_two<f32>>(arr);
  let summed = array_reduce<f32, f32, sum<f32>>(0.0);
}
```

[^1]: https://www.w3.org/TR/WGSL/#array-types
[^2]: https://www.w3.org/TR/WGSL/#array-builtin
[^3]: https://www.w3.org/TR/WGSL/#arrayLength-builtin
