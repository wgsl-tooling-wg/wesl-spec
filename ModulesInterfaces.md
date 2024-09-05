# Modules

## Summary

We propose adding a module signature mechanism to the WGSL shading language as an extension.

## Assumptions

Assumes that [`load`](./Imports.md) and [Modules](./Modules.md) has already been implemented.

# Motivation

Encapsulation and hiding is an important part of being a responsible library maintainer while still allowing for 
the evolution and refactoring of implementation details. Currently WESL does not have a mechanism for visibility and so 
assumes everything is public by default.

Additionally we have seen examples of WGSL code that share the same interface but different implementations. One example 
that we've seen was for a reduction algorithm that ran on the GPU. This algorithm needed to combine two values but the exact method for combining the values depended on the end user's requirements. Module signatures would allow these operations and even user defined ones to be type checked. The ability to check module implementations against their signatures would also prove very beneficial to type constraints in [generic modules](./GenericModules.md) and [generic functions](./GenericFunctions.md).

# Guide-level explanation

A module signature declared using `mod sig` keyword. A module signature contains a set of declarations of elements such as functions, variables, modules, and constants without specifying their implementation. A module can be said to "return" such a signature, which signals to type checkers that only the symbols defined in the interface should be exposed, and that all the symbols need to be present for typechecking to pass.

Below is an example of a module signature and its usage:

```wgsl
// A module signature:

mod sig MathImpl {
    const DEG_TO_RAD: f32;
    fn quat_from_euler(xyzRad: vec3<T>) -> vec4<f32>;
}

// Another module signature:
mod sig MainMath {
    mod Float -> MathImpl;
}

// This can be type checked to determine 
// whether Math (and consequently Float conform to the provided signature)
mod Math -> MainMath {
    mod FloatMath {
        const DEG_TO_RAD: f32 = 0.0174533;
        fn quat_from_euler(xyzRad: vec3<f32>) -> vec4<f32> {
            // Implementation here
        }

        fn private_function() -> f32 {
            // Implementation here
        }
    }
    // An alias for a module
    mod Float = FloatMath;
}

// Usage
Math::Float::quat_from_euler(vec4<f32>(90.0 * Math::Float::DEG_TO_RAD))


// Not allowed as the FloatMath module isn't in the interface
Math::FloatMath::quat_from_euler(vec4<f32>(90.0 * Math::Float::DEG_TO_RAD))

// Not allowed as private_function isn't in the interface
Math::Float::private_function()
```

# Reference-level explanation

Module signatures are parsed as follows, with spaces and comments allowed between tokens:

```bnf

module_sig_decl :
  attribute * 'mod' 'sig' "{" global_sig * "}" ";"?
;

global_sig :
  | function_sig
  | nested_mod_sig
  | global_variable_sig
  | global_value_sig
;

function_sig : 
  attribute * function_header ';'

nested_mod_sig : 
  attribute * 'mod' ident '->' type_specifier  ';'
;

global_variable_sig :
  attribute * variable_decl ? ';'
;

global_value_sig :
  'const' ident ( ':' type_specifier )  ';'
| attribute * 'override' ident ( ':' type_specifier )  ';'
;

extend module_member_decl : 
    | module_sig_decl
;

module_decl:  
  attribute * 'mod' ident ('->' type_specifier)? "{" module_member_decl * "}" ";"?
;

module_alias_decl :
  attribute * 'mod' optionally_typed_ident '=' module_path ';'
;

```

Where `ident`, `attribute`, `type_specifier`, `function_header` and `optionally_typed_ident` are defined in the WGSL grammar. The WGSL grammer is additionally extended as follows: 

```bnf
extend global_decl :
  | module_sig_decl
;
```

## Linking WESL Files

Linking WESL files with modules signatures is very similar to the [Modules](./Modules.md) linking algorithm. There is just 
one addition:

- Module signatures should be removed from output code

## Type Checking

In general, type checking in WESL is to be performed by testing for structural equality. 

This means that given a module signature and a module, the following should hold after the canonicalisation step:


- all types declared in the module signature should be present either as an alias or a struct declaration. 
- all submodules declared in the signature should be present and be structurally a superset of the specified submodule type
- all functions should be present and have the same type and number of arguments as well as the same return type. 
- all constants, overrides and vars should be present and have the same types. 
- each of these respective elements should have identical attributes values present. 
- additional symbols are ignored and should be treated as private after assignment or declaration. 

## Drawbacks

Are there reasons as to why we should we not do this?

- This introduces additional complexity to the specification.
- Requires that users understand additional syntax
- While not strictly required, a typechecker is pratically required in order to validate signatures.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

## Alternatives 

### Visibility modifiers 

We could introduce explicit visibility modifiers rather than interfaces to mark a particular symbol as public/private.
In one sense this may be simpler, however it would require us to modify parsing of quite a large surface area. 
Visibility would also not be a feature that could be later extended to offer more powerful abstractions.

### No concept of visibility 

We could continue without visibilty or module signatures. The downside is relying on duck typing if/when generics are implemented could result in a worse experience, and type inference would be a more complicated approach.


