# Conditional Compilation

## Overview

> This section is non-normative

Conditional compilation is a mechanism to modify the source code based on parameters passed to the compiler.
This specificaton extends the [*attribute* syntax](https://www.w3.org/TR/WGSL/#attributes) with three new attributes: `@if`, `@elif`, `@else`.
These attributes indicate that the syntax node they decorate can be removed by the compiler based on feature flags.

> NOTE: This implementation is similar to the `#[cfg(feature = "")]` syntax in rust.

### Usage Example

```rs
// global variables and bindings...
@if(textured)
@group(0) @binding(0) var my_textuer: texture_2d<f32>;

// structs declarations and struct members...
struct Ray {
  position: vec4f,
  direction: vec4f,
  @if(debug_mode && raytracing_enabled)
  ray_steps: u32,
}

// function declarations, parameters and statements...
fn main() -> vec4 {
  @if(legacy_implementation || (is_web_version && xyz_not_supported))
  let result = legacy_impl();
  @else
  let result = modern_impl();
}
```

## Definitions

* **Compile-time expression**: A *compile-time expression* is evaluated by the compiler before the rest of the program. It lives in the *compile-time scope*.
 Its grammar is a subset of normal WGSL [expressions](https://www.w3.org/TR/WGSL/#expressions). it must be one of:
 * a *compile-time feature*,
 * a [logical expression](https://www.w3.org/TR/WGSL/#logical-expr): logical not (`!`), short-circuiting AND (`&&`), short-circuiting OR (`||`),
 * a [parenthesized expression](https://www.w3.org/TR/WGSL/#parenthesized-expressions),
 * a boolean literal value (`true` or  `false`).

* **Compile-time scope**: The *compile-time scope* this is an independent scope from the [*module scope*](https://www.w3.org/TR/WGSL/#module-scope), meaning it cannot see any declarations from the source code, and its identifers are independent.

* **Compile-time feature**: A *compile-time feature* is an identifier that evaluates to a boolean. It is set to `true` if the feature is *enabled* during the compilation phase.

* **Compile-time attribute**: A *compile-time attribute* is parametrized by a *compile-time expression*. It is eliminated after the compilation phase but can affect the syntax node it decorates.

## Location of *Compile-time attributes*

A compile-time attribute can appear in all places where the WGSL grammar allows attributes, except if removal of the decorated syntax node would lead to syntactically incorrect code.
The grammar is extended to allow attributes in several locations previously not allowed by the WGSL grammar. These extensions are specified in section [updated grammar](#updated-grammar).

### Summary

1. The grammar is extended to allow *compile-time attributes* before the following syntax nodes:
  * const value declarations
  * variable declarations
  * directives
  * switch clauses
  * all statements that did not allow attributes:
    * assignment statements
    * increment and decrement statements
    * control flow statements (if/else if/else, switch, loop, for, while, break, continue, continuing, return, discard)
    * function call statements
    * const assertion statements

2. A Compile-time attribute CAN decorate the following syntax nodes:
  * structure declarations
  * structure members
  * function declarations
  * function formal parameter declarations
  * variable declarations and value declarations
  * all statements, except those disallowed
  * directives

3. A Compile-time attribute CANNOT decorate the following syntax nodes, even if the WGSL grammar allows attributes before these syntax nodes:
* function return types
* the body (part surrounded by curly braces) of:
  * function declarations
  * switch statements
  * switch clauses
  * loop statements
  * for statements
  * while statements
  * if/else statements

## `@if` attribute

A new *compile-time attributes* is introduced.
* An `@if` attribute takes a single parameter. It marks the decorated node for removal if the parameter evaluates to `false`.

> NOTE: See the [possible future extensions](#possible-future-extensions) for the attributes `@elif` and `@else`.
> They may be introduced in the specification in a future version if deemed useful.

*Example*

```rs
@if(feature_1 && (!feature_2 || feature_3))
fn f() { ... }
@if(!feature_1)                               // corresponds to @elif(!feature_1)
fn f() { ... }
@if(feature_1 && !(!feature_2 || feature_3))  // corresponds to @else
fn f() { ... }
```

## Execution of the conditional compilation phase

The conditional compilation phase *should* be the first phase to run in the full WESL compilation pipeline.

> NOTE: The conditional compilation was designed to be incremental. In case some features can only be resolved at runtime, the compiler can be invoked in two passes:
> The first pass is invoked with compile-time features and returns a partially compiled WESL source. The second pass is invoked with the runtime features and returns a fully compiled WGSL source.

1. The compiler is invoked with a list of features to *enable* or *disable*.
2. The source file is parsed.
3. The *compile-time features* in *Compile-time expressions* are resolved:
    * If the feature is not present in the compiler's feature list, it is unchanged.
    * If the feature is *enabled*, the identifier is replaced with `true`.
    * If the feature is *disabled*, the identifier is replaced with `false`.
   
   The expression is *fully resolved* if all its *compile-time features* have been replaced by `true` or `false`.
5. *Compile-time attributes* which have *fully resolved expressions* are evaluated:
    * If the syntax node is marked for removal: it is eliminated from the source code along with the attribute.
    * Otherwise, if all features in only the attribute is eliminated from the source code.
4. The updated source is passed to the next compilation phase. (e.g. import resolution)

## Possible future extensions

> This section is non-normative

* `@else` and `@elif` attributes:
  * An `@elif` attribute decorates the next sibling of a syntax node decorated by a `@if` or an `@elif`. It takes one parameter.
     It marks the decorated node for removal if its parameter evaluates to `true` AND if the previous `@if` and `@elif` attribute parameters evaluate to false.
  * An `@else` attribute decorates the next sibling of a syntax node decorated by a `@if` or an `@elif`. It does not take any parameter.
     It marks the decorated node for removal if the previous `@if` and `@elif` attribute parameters evaluate to false.

The `@else` attribute has the nice property that all cases lead to generating a node, and *could* therefore be used in places where the node is required (e.g. expressions)

*Example*

```rs
@if(feature_1 && (!feature_2 || feature_3))
fn f() { ... }
@elif(!feature_1)
fn f() { ... }
@else
fn f() { ... }
```

* High-complexity *compile-time expressions*: if we end up implementing other compile-time attributes, such as loops (e.g. `@for`, `@repeat`), or [compile-time-evaluable](https://zig.guide/language-basics/comptime/) expressions, then we would need to extend the grammar of *compile-time expressions*. It would also affect this proposal.

*Example*

```rs
@comptime
fn response_to_the_ultimate_question() -> u32 {
  return 42;
}

@if(response_to_the_ultimate_question() == 42)
fn f() { ... }
```

* Decorating other WESL language extensions: import statements could be decorated with *compile-time attributes* too.

*Example*

```rs
@if(use_bvh)
import accel/bvh_acceleration_structure as scene_struct;
@else
import accel/default_acceleration_structure as scene_struct;
```

## Appendix: Updated grammar

(TBD)

    variable_or_value_statement :
       variable_decl
     | variable_decl '=' expression
     | attribute * 'let' optionally_typed_ident '=' expression
     | attribute * 'const' optionally_typed_ident '=' expression


    variable_decl :
     attribute * 'var' _disambiguate_template template_list ? optionally_typed_ident


    global_value_decl :
       attribute * 'const' optionally_typed_ident '=' expression
     | attribute * 'override' optionally_typed_ident ( '=' expression ) ?

