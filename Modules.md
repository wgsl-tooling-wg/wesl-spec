# Modules

## Summary

We propose adding a module system mechanism to the WGSL shading language as an extension.

## Assumptions

Assumes that [`load`](./Imports.md) has already been implemented.

# Motivation

Currently all symbols in wgsl share a single global namespace with a prohibition against symbols with the same name. Many existing implementations of imports (not unlike [our proposal](./Imports.md)) simply add to this global scope. For relatively self contained projects this is not a problem, however when considering a broader ecosystem of packages containing reusable shaders, collisions become much more likely. 

Additionally WGSL provides very little way to encapsulate and organise code. This proposal would pave the way for both
[Module Interfaces](./ModulesInterfaces.md) and [Generic Modules](./GenericModules.md). The former would allow a graphics programmer to optionally control the visibility and typecheck the symbols within a module, while the latter would allow for more reusable code and the ability to build powerful abstractions. 

Finally [Includes](./Include.md), another extension of modules, gives a way to compose behaviour in a manner not too dissimilar to inheritance, which is important for use cases such as writing extended shaders based on Standard PBR workflows in game engines such as [Bevy](https://bevyengine.org/). 

# Guide-level explanation

A module is declared using the `mod` keyword. A module contains a set of declarations such as structs, functions, aliases, variables, and constants. A module may also contain another module inside it. Modules however are not allowed to contain load statements as the semantics of this behaviour would be confusing. Declarations within a module are accessed using the module name and the `::` operator. 

There are two ways to declare a module - as an alias for another module, or as an inline module. Here is an example of both forms, along with a module usage example:

```wgsl
// An inline module:
mod Math {
    // Another inline module:
    mod FloatMath {
        const DEG_TO_RAD: f32 = 0.0174533;

        fn quat_from_euler(xyzRad: vec3<f32>) -> vec4<f32> {
            // Implementation here
        }
    }

    // An alias for a module
    alias Float = FloatMath;
}

// Usage
Math::Float::quat_from_euler(vec4<f32>(90.0 * Math::Float::DEG_TO_RAD))
```

Libraries are automatically wrapped in a root module based on their name declared in the WESL manifest. This is to further
avoid namespace pollution. This has the downside that it is not possible to use entrypoints declared within a third party shader library unless the [`include`](./Include.md) feature is available and used to add symbols from within a library into the global namespace.

# Reference-level explanation

Module statements are parsed as follows, with spaces and comments allowed between tokens:

```bnf
module_decl_main: 
    | module_decl
;

module_member_decl :
  ';'
| global_variable_decl ';'
| global_value_decl ';'
| type_alias_decl ';'
| struct_decl
| function_decl
| const_assert_statement ';'
| module_decl_main
;

module_decl:  
  attribute * 'mod' ident "{" module_member_decl * "}" ";"?
;

module_path :
  ident ('::' ident)*
;

```

Where `ident` is defined in the WGSL grammar. The WGSL grammer is additionally extended as follows: 

```bnf
extend global_decl :
  | module_decl_main
;

template_elaborated_ident :
  (module_path '::')? ident _disambiguate_template template_list ?
```

## Behaviour-changing items

These items cannot be imported, but they affect behavior:

- [Extensions](https://www.w3.org/TR/WGSL/#extensions)
- [Global diagnostic filters](https://www.w3.org/TR/WGSL/#global-diagnostic-directive)

These are always set at the very top in the main module, and affect all loaded files. They come before the imports.

When, during parsing of loaded files and modules, we encounter usage of a module that has an extension or a global diagnostic filter within it, we check if the main module enables it, or sets the filter.

If yes, everything is fine. If not, we throw an error.

## Bundling WESL Files

As modules are imported into the global namespace, a naive bundling algorithm would be:
- Remove all `load` statements
- Concatenate files together according to the linking logic described in [Imports](./Imports.md) 

## Linking WESL Files

Linking WESL files with modules is a little bit more complex than the base [Imports](./Imports.md) linking algorithm.

To do so the following steps must be taken:
- Bundling needs to occur prior to linking. 
- A map of modules to their implementation should be made. A canonical module name should be picked using the logic described in the Name Mangling section in this proposal. 
- Module usages should be replaced with the canonical module name.
- Next symbols within the module need to be mangled using the canonical module name using the [mangling algorithm](./NameMangling.md) and should be added to the global scope. 
- Usage of symbols within a module should then replaced with the mangled representation using the same algorithm. 
- Finally a plain wgsl module is produced

### Name Mangling
Symbols within a module are mangled based on the module path, using the scheme described in [Name Mangling](./NameMangling.md). 

As modules can be aliased, the module path choosen by the mangling scheme uses the following algorithm:  
 - Picks the path with the fewest number of parts in it
    - If this is a tie, picks the path string with the smallest number of unicode characters in it
        - Finally if that is also a tie, uses a region invariant string comparision function, and chooses the path that is alphabetically first (inclusive of the `::` operators)

This scheme means that if a given symbol has been included into the global scope, it is not mangled at all. It has the absolute minimum number of parts in the path!

An example of how to apply the algorithm described above, would be the name used in mangling the `quat_from_euler` function. The name chosen to be mangled would be `Math::Float::quat_from_euler`, rather than `Math::FloatMath::quat_from_euler`. This is because while they have the same number of parts in their paths, `Math::Float::quat_from_euler` has fewer number of characters.

## Drawbacks

Are there reasons as to why we should we not do this?

- This introduces a major new syntactical element which may later conflict with the emerging WGSL standard.
- Requires that users understand the module syntax which may be difficult for object oriented programmers
- Can be quite verbose though module names can be shortened using module aliases (or `include` if adopted)
- Requires that the [`include`](./Include.md) proposal be adopted if third party modules are to be allowed to define entrypoints and bindings.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

## Alternatives 

### Extending Structs

We could namespace symbols within the context of a struct. However as so many shaders deal with primitive types and don't adopt an OO approach, it may be unwise to tie these two concerns together.

### Files as modules

We could make every file also be a module. This has some downsides, in that entrypoints either have to be mangled or exposed in some way to the WebGPU API which may leak details of mangling to the user. Additionally, mixing the concept of files and modules may introduce confusion and complexity if we later added a module or trait like construct. 

## What is the impact of not doing this?

A number of interesting language features (described in the motivation section) depend on modules. While all of them may not be adopted, modules seem to be a fruitful abstraction for future work in adding necessary features to the language.
