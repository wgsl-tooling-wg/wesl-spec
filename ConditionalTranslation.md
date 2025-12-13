# Conditional Translation and Shader Specialization

## Overview

> *This section is non-normative*

Conditional translation and shader specializaton are two mechanisms to control the WGSL code produced based on parameters passed to the *WESL translator*.
This specification extends the [*attribute* syntax](https://www.w3.org/TR/WGSL/#attributes) with new attributes: `@param` defines *module parameters* for shader specializaton via constant injection, while `@if`/`@elif`/`@else` control conditional translation.

> [!NOTE]
> Compared to Rust, `@param const my_param = true` is analogous to a feature flag, while `@if(my_param)` is analogous to `#[cfg(feature = "my_param")]`.

### Usage Example

```wgsl
// define module parameters with default values...
@param const TEXTURED = true;
@param const DEBUG = false;
@param const RAYTRACING_ENABLED = false;
@param const LEGACY_COMPAT = false;

// global variables and bindings...
@if(TEXTURED)
@group(0) @binding(0) var my_texture: texture_2d<f32>;

// structs declarations and struct members...
struct Ray {
  position: vec4f,
  direction: vec4f,
  @if(DEBUG && RAYTRACING_ENABLED)
  ray_steps: u32,
}

// function declarations, parameters and statements...
fn main() -> vec4 {
  @if(LEGACY_COMPAT)
  let result = legacy_impl();
  @else
  let result = modern_impl();
}
```

Quirky examples

```wgsl
// attribute order does not matter.
@compute @if(FEATURE) fn main() { }
// ... is equivalent to this (preferred)
@if(FEATURE) @compute fn main() { }
```

## Definitions

* A **module parameter** is an module-scope const-declaration decorated with the `@param` attribute. Function-scope const-declarations cannot be module parameters*.
* A **condition expression** is evaluated by the *WESL translator* and eliminated after translation.
  Its grammar is a subset of normal WGSL [expressions](https://www.w3.org/TR/WGSL/#expressions). it must be one of:
  * a [literal value](https://www.w3.org/TR/WGSL/#literals) (boolean or numeric literal).
  * an [identifier expression](https://www.w3.org/TR/WGSL/#value-identifier-expr) referring to a *module parameter* in scope,
  * a [logical expression](https://www.w3.org/TR/WGSL/#logical-expr): logical not (`!`), short-circuiting AND (`&&`), short-circuiting OR (`||`),
  * a [comparison expression](https://www.w3.org/TR/WGSL/#comparison-expr): `==`, `!=`, `<`, `<=`, `>`, `>=`,
  * a [parenthesized expression](https://www.w3.org/TR/WGSL/#parenthesized-expressions),
* **condition attributes** (`@if`, `@elif` and `@else`) are eliminated after translation but can also eliminate the syntax node it attached to.

## Location of *condition attributes*

A *condition attribute* can appear before the following syntax nodes:

* [directives](https://www.w3.org/TR/WGSL/#directives)
* [variable and value declarations](https://www.w3.org/TR/WGSL/#var-and-value)
* [type alias declarations](https://www.w3.org/TR/WGSL/#type-alias)
* [const assertions](https://www.w3.org/TR/WGSL/#const-assert-statement)
* [function declarations](https://www.w3.org/TR/WGSL/#function-declaration-sec)
* function formal parameter declarations
* [structure declarations](https://www.w3.org/TR/WGSL/#struct-types)
* structure member declarations
* [statements](https://www.w3.org/TR/WGSL/#statements)
* [switch clauses](https://www.w3.org/TR/WGSL/#switch-statement)

> [!NOTE]
> *Condition attributes* are not allowed in places where removal of the syntax node would lead to syntactically incorrect code. The current set of *condition attribute* locations guarantees that the code is syntactically correct after specialization. This is why *condition attributes* are not allowed before expressions.

### Update to the WGSL grammar

The WGSL grammar allows attributes in several locations where *condition attributes* are not allowed (1). Conversely, the WGSL grammar does not allow attributes in several locations where *condition attribute* are allowed (2).

Refer to the [updated grammar appendix](#appendix-updated-grammar) for the list of updated grammar non-terminals.

1. A *condition attribute* CANNOT decorate the following syntax nodes, even if the WGSL grammar allows attributes before these syntax nodes:
   * function return types
   * the body (part surrounded by curly braces) of:
     * function declarations
     * switch statements
     * switch clauses
     * loop statements
     * for statements
     * while statements
     * if/else statements
     * continuing statements

2. The grammar is extended to allow *condition attributes* before the following syntax nodes:
   * const value declarations
   * variable declarations
   * directives
   * struct declarations
   * switch clauses
   * assignment statements
   * increment and decrement statements
   * break statements
   * break-if statements
   * continue statements
   * continuing statements
   * return statements
   * discard statements
   * function call statements
   * const assertion statements

> [!WARNING]
> We added attributes on assignment and increment/decrement statements. This change makes the WESL grammar no longer [LR(1)](https://en.wikipedia.org/wiki/LR_parser). See [`wesl-rs#162`](https://github.com/wgsl-tooling-wg/wesl-rs/issues/162) for details.
>
> Due to this limitation, *wesl-rs* does not allow attributes on assignment, increment and decrement statements starting with a `(`. Concretely, this is not supported by *wesl-rs*: `@if(FOO) (x)++`.

## `@param` attribute

A module-scope const-declaration can be decorated with the `@param` attribute to turn it into a *module parameter*.
*Module parameters* work like any other const declarations, but also provide two advantages:
* their value can be overriden by the *WESL translator*.
* they can be used in *condition expressions* if they are of type boolean.

(TODO)
*Module parameter* overrides for direct dependencies can be set from the `wesl.toml` manifest file.
The manifest file can override *module parameters* by providing either a literal automatically convertible to the *module parameter* type, or the *fully-qualified path* to a *module parameter* in the current package.
Thus, the override in the dependency is "bound" to a new override in the current pacakge.

(TODO)
*Module parameter* overrides for the root package, whoever, are passed to the *WESL translator* at *shader-translation-time*.
The *WESL translator* refers to a *module parameter* using the *fully-qualified path* of its declaration.
It can override the value by providing a literal automatically convertible to the *module parameter* type.

(TODO)
> [!NOTE]
> Shader libraries have no means of overriding their own *module parameters*.

(TODO)
If a dependency appears several times in the dependency graph, overrides several times the same *module parameter* (even if overriden with the same value), that dependency cannot be *unified*.
Otherwise, the dependency can be unified, and all overrides apply to the unified version.

## `@if`, `@elif` and `@else` attributes

The `@if` attribute takes a single *condition expression* parameter. 
It marks the decorated node for removal if its *condition expression* evaluates to `false`.

The `@elif` attribute takes a single *condition expression* parameter. 
It can only be attached to the next sibling of a syntax node decorated by a `@if` or an `@elif`. 
It marks the decorated node for removal if its *condition expression* evaluates to `false` OR any of if the previous `@if` and `@elif` attribute parameters in the chain evaluate to `true`.

The `@else` attribute takes no parameter. 
It can only be attached to the next sibling of a syntax node decorated by a `@if` or an `@elif`.
It marks the decorated node for removal if any of the previous `@if` and `@elif` attribute *condition expressions* in the chain evaluate to `true`.
    
A syntax node may at most have a single `@if`, `@elif` or `@else` attribute. Checking for multiple features is done with an AND (`&&`) expression.

Example:

```wgsl
@if(PARAM1 && PARAM2) const decl: u32 = 0;

@if(PARAM1 && (!PARAM2 || PARAM3))
fn f() { ... }
@elif(!PARAM1)   // identical to @if(!PARAM1)
fn f() { ... }
@else            // identical to @if(!(!PARAM2 || PARAM3))
fn f() { ... }
```

## Error cases

It is a translate-time error if:

* The *WESL linker* was invoked with a *shader parameter* override that does not exist;
* The type of a *shader parameter* override does not match the type of the declaration;
* An identifier used in a *condition expression* does not reference a boolean shader parameter.
* A syntax node has multiple `@param`, `@if`, `@elif` or `@else` attributes. There can be at most one of these four attributes per node.
  In particular, a module parameter cannot be have a condition attribute.

## Execution of the conditional translation phase

1. The *WESL translator* is invoked with a list of *module parameters* names and values to override *for each package*.
2. The source file is parsed.
3. For each overriden *module parameter*, the const-declaration initializer are replaced with the overridden value and the `@param` attribute is removed.
4. *condition expressions* are evaluated in order.
  * If it evaluates to `false`, the decorated syntax node is eliminated from the source code along with the attribute.
  * If it evaluates to `true`, the attribute is removed and subsequent nodes decorated with `@elif` or `@else` are eliminated.
6. The updated source code is passed to the next translation phase. (e.g. import resolution)

### Incremental translation

In case some features can only be resolved at runtime, a *WESL translator* can *optionally* support specialization in multiple passes.
In the initial passes, the *WESL translator* is invoked with some *shader parameters* marked explicitly to be preserved.

## Possible future extensions
> *This section is non-normative*

* Decorating other WESL language extensions: import statements could be decorated with *condition attributes* too.

  Example:
  ```wgsl
  @if(USE_BVH)
  import accel::bvh_acceleration_structure as scene_struct;
  @else
  import accel::default_acceleration_structure as scene_struct;
  ```

## Appendix: Summary of constant declarations

There are three types of so-called "constants" in WGSL. They are evaluated at
different shader stages.

| syntax | declaration kind | shader stage  | module-scope | function-scope | `@if` | array element count | 
| ------ | ---------------- | ------------- | ------------ | -------------- | ----- | ------------------- |
| `@param const x = 0;` | module parameter              | 1 shader-translation | yes | no  | no  | yes |
| `const x = 0;`        | const-declaration             | 2 shader-creation    | yes | yes | yes | yes |
| `override x = 0;`     | pipeline-overridable constant | 3 pipeline-creation  | yes | no  | yes | no  |

## Appendix: Updated grammar
The following non-terminals are added or modified:

```grammar
    diagnostic_directive :
      attribute * 'diagnostic' diagnostic_control ';'

    enable_directive :
      attribute * 'enable' enable_extension_list ';'

    requires_directive :
      attribute * 'requires' software_extension_list ';'

    struct_decl :
      attribute * 'struct' ident struct_body_decl
     
    type_alias_decl :
      attribute * 'alias' ident '=' type_specifier

    variable_or_value_statement :
      variable_decl
    | variable_decl '=' expression
    | attribute * 'let' optionally_typed_ident '=' expression
    | attribute * 'const' optionally_typed_ident '=' expression

    variable_decl :
      attribute * 'var' _disambiguate_template template_list ? optionally_typed_ident
     
    global_variable_decl :
      variable_decl ( '=' expression ) ?

    global_value_decl :
      attribute * 'const' optionally_typed_ident '=' expression
    | attribute * 'override' optionally_typed_ident ( '=' expression ) ?

    case_clause :
      attribute * 'case' case_selectors ':' ? compound_statement

    default_alone_clause :
      attribute * 'default' ':' ? compound_statement

    assignment_statement :
      attribute * lhs_expression ( '=' | compound_assignment_operator ) expression
    | attribute * '_' '=' expression

    increment_statement :
      attribute * lhs_expression '++'

    decrement_statement :
      attribute * lhs_expression '--'

    break_statement :
      attribute * 'break'

    break_if_statement :
      attribute * 'break' 'if' expression ';'

    continue_statement :
      attribute * 'continue'
     
    continuing_statement :
      attribute * 'continuing' continuing_compound_statement

    return_statement :
      attribute * 'return' expression ?
    
    discard_statement:
      attribute * 'discard'

    func_call_statement :
      attribute * call_phrase

    const_assert_statement :
      attribute * 'const_assert' expression

    statement :
      ';'
    | return_statement ';'
    | if_statement
    | switch_statement
    | loop_statement
    | for_statement
    | while_statement
    | func_call_statement ';'
    | variable_or_value_statement ';'
    | break_statement ';'
    | continue_statement ';'
    | discard_statement ';'
    | variable_updating_statement ';'
    | compound_statement
    | const_assert_statement ';'

    attribute :
      '@' ident_pattern_token argument_expression_list ?
    | align_attr
    | binding_attr
    | blend_src_attr
    | builtin_attr
    | const_attr
    | diagnostic_attr
    | group_attr
    | id_attr
    | interpolate_attr
    | invariant_attr
    | location_attr
    | must_use_attr
    | size_attr
    | workgroup_size_attr
    | vertex_attr
    | fragment_attr
    | compute_attr
    | if_attr

    if_attr:
      '@' 'if' '(' expression ',' ? ')'
```
