# Conditional Compilation

## Overview

Conditional compilation is a mechanism to modify the source code based on parameters passed to the compiler. We distinguish two kinds:

 * **unstructured**: arbitrary code sections can be injected or conditionaly included. This is what the C preprocessor and macros do, and a lesser version of that is proposed in [`simple templating`](SimpleTemplating.md). 
 * **structured**: only structural elements of the syntax can be manipulated, e.g. a whole declaration, a member, etc. Rust uses this approach with the `#[cfg]` attribute.

We think that *structured* is the way to go. It leads to clearer and safer code, which is more important than implementation complexity in our eyes.
Also, a language that has a good expressive power already should not need a way to hack around with arbitrary code injection, like C macros do.

| unstructured | structured |
|--------------|------------|
| (+) much easier to implement: just look for the `#word` symbols | (-) harder to implement: requires a full parsing |
| (+) often language-agnostic (see the [C preprocessor](https://en.wikipedia.org/wiki/C_preprocessor). Familiar to C developers. | (-) a new syntax needs to be taught, not always self-explanatory |
| (-) poorly integrated in the language, harder to read by humans | (+) is a well-designed syntax feature of the language |
| (+) technically more expressive (e.g. [manipulating identifiers](https://en.wikipedia.org/wiki/C_preprocessor#Token_concatenation)) | (-) can only conditionaly include parts of the syntax tree, poor text generation capabilities. |
| (-) behaves unpredictably, intent and behavior is hidden | (+) intent and behavior is visible at the usage site |
| (-) no type checking, no syntax checking, because code is generated dynamically | (+) can statically check that conditional variants lead to syntactically correct and type-valid code |
| (-) poor IDE support | (+) IDE can check all possible code paths |

## Proposal

We propose to leverage the [*attribute* syntax](https://www.w3.org/TR/WGSL/#attributes) to decorate syntax nodes with conditional compilation attributes.
The compiler is provided with a set of '*compile-time features*' to enable (see below).

### Definitions

 * **Compile-time expression**: A *compile-time expression* is evaluated by the compiler before the rest of the program. It lives in the *compile-time scope*.

 * **Compile-time scope**: The *compile-time scope* this is an independent scope from the [*module scope*](https://www.w3.org/TR/WGSL/#module-scope), meaning it cannot see any declarations from the source code, and its identifers are independent.

 * **Compile-time feature**: A *compile-time feature* is an identifier that evaluates to a boolean. It is set to `true` if the feature is *enabled* during the compilation phase.

 * **Compile-time attribute**: A *compile-time attribute* is parametrized by a *compile-time expression*. It is eliminated after the compilation phase but can affect the syntax node it decorates.

### Location of *Compile-time attributes*

(TODO: this needs more reflexion)

A Compile-time attribute CAN decorate the following syntax nodes:
 * structure declarations
 * structure members
 * function declarations
 * ?? function formal parameter declarations ?? (is this needed?)
 * variable declarations and value declarations
 * all statements, except those disallowed below
 * directives

Because eliminating the decorated node would lead to invalid code, a Compile-time attribute CANNOT decorate the following syntax nodes:
 * bodies of function declarations, switch statements, switch clauses, loop statements, for statements, while statements, if/else statements
 * function return types
 
> Note: The WGSL grammar currently does not allow attributes in front of const value declarations, variable declarations, directives, switch clauses and several statements. We would extend the syntax to allow them.
> Which exacly we will enable is subject to discussion. Enabling them before all statements is perhaps undesirable.

### Syntax 1: Features only and the `@ifdef`, `@ifndef` attributes

> This is the least complex implementation.

A *compile-time expression* must be a single *compile-time feature*.

Two new attributes are introduced that take a single parameter.
* The `@ifdef` attribute marks the decorated node for removal if the parameter evaluates to `false`.
* The `@ifndef` attribute marks the decorated node for removal if the parameter evaluates to `true`.

*Example*

```rs
@ifdef(my_feature)
fn f() { ... }
@ifndef(my_feature)
fn f() { ... }
```

### Syntax 2: `@if` attribute with compile-time logical expressions

> This is the medium-complexity implementation.

A *compile-time expression* is a subset of normal WGSL [expressions](https://www.w3.org/TR/WGSL/#expressions). it must be one of:
 * a *compile-time feature*,
 * a [logical expression](https://www.w3.org/TR/WGSL/#logical-expr): logical not (`!`), short-circuiting AND (`&&`), short-circuiting OR (`||`),
 * a [parenthesized expression](https://www.w3.org/TR/WGSL/#parenthesized-expressions).

Three new attributes are introduced.
* An `@if` attribute takes a single parameter. It marks the decorated node for removal if the parameter evaluates to `false`.
* An `@elif` attribute decorates the next sibling of a syntax node decorated by a `@if` or an `@elif`. It takes one parameter.
   It marks the decorated node for removal if its parameter evaluates to `true` AND if the previous `@if` and `@elif` attribute parameters evaluate to false.
* An `@else` attribute decorates the next sibling of a syntax node decorated by a `@if` or an `@elif`. It does not take any parameter.
   It marks the decorated node for removal if the previous `@if` and `@elif` attribute parameters evaluate to false.

*Example*

```rs
@if(feature_1 && (!feature_2 || feature_3))
fn f() { ... }
@elif(!feature_1)
fn f() { ... }
@else
fn f() { ... }
```

### Execution of the conditional compilation phase

The conditional compilation phase *should* be the first phase to run in the full WESL compilation pipeline.

1. The compiler is invoked with a list of features to enable.
2. The source file is parsed.
3. *Compile-time attributes* are evaluated
    * If the syntax node is marked for removal: it is eliminated from the source code along with the attribute.
    * Otherwise, only the attribute is eliminated from the source code.
4. The updated source is passed to the next compilation phase. (e.g. import resolution) 

### Possible future extensions

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
