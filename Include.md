# Modules

## Summary

We propose adding a module extension mechanism to the WESL shading language.

## Assumptions

Assumes that [`load`](./Imports.md), [Modules](./Modules.md) and [Module Interfaces](./ModulesInterfaces.md) have already been implemented. The last one however isn't essential to this proposal which could be tweaked to function without it.

# Motivation

While WESL currently provides a way to organize and encapsulate types, functions and state. However it does *not* 
provide a way to specialize behaviour, or in other words does not provide a mechanism for inheritance. 

Specialization is important for a number cases, an important and frequent one of which is for abstracting the implementation and configuration details of Physically Based Rendering (PBR), reducing the boilerplate and number of edge cases a technical artist has to consider. 

At its core, PBR is a mathematical parametrisation of the interaction of an infintesimally small fragment of a physical materials with light. For example many implementations of PBR allow one to specify the physical qualities of a surface such as the albedo, normal, how rough or smooth a surface is and whether it is metallic or not. These qualities are used in a complex integral that is used to drive the appearance of a surface. However most workflows do not require
technical artists to know the details of these integrals, and instead allow them to author materials by simply varying 
the PBR values across the surface. 

An example of how we would want such our abstraction to look like in a game engine such as [Bevy](https://bevyengine.org/)
would be as follows:

```wgsl
// Including standard material adds all the symbols 
// to the global module including the vertex and fragment entry points. 
include BevyPbr::StandardMaterial;

// Tells the linker to resolve calls to `resolve_pbr_inputs`
// to this function instead of the base. This is constrained to the 
// current namespace; in this case the namespace is the global scope.
patch fn resolve_pbr_inputs(pbr_inputs: &PbrInputs) {
  // Calls the base method, populating the pbr inputs with the default values
  BevyPbr::StandardMaterial::resolve_pbr_inputs(pbr_inputs); 
  
  // Insert code to procedurally generate a carbon fibre coating
}
```

This is quite a concise representation and a big improvement in usability over what is available in Bevy today.

# Guide-level explanation

This proposal introduces two mechanisms which work in concert to specialize behaviour. These are `include` and `patch`. 

One could practically think of `include` as copying the contents of a given module into the current namespace. This serves dual purposes. Firstly it behaves as a means of not having to refer to a symbol within a deeply nested module by its fully qualified name, reducing verbosity. Secondly it allows one to inherit the behaviour of another module (commonly also called a mixin). It is possible to use `include` multiple times in a single namespace, provided no names in the scope clash.  Namespaces which use `includes` do not have privileged access to the symbols within a module, so the same visibility rules apply. 

This is just one half of the story however, as the module being included is not aware of the namespace doing the including. Therefore we also need a way of specializing behaviour in functions within the target namespace to bridge the gap between the two. This is where `patch` comes in. `patch` is a keyword that may be applied to a function declaration. It instructs the linker to replace usages of the specified function within the namespace with the newly declared function. The patched function may still be invoked using its fully qualified name. Naturally `patch` comes with the restriction that function signatures have to match. 

A possible extension to this specification would be to only allow `virtual` functions to be patched.

# Reference-level explanation

`include` is parsed as follows, with spaces and comments allowed between tokens:

```bnf
include_decl :
  'include' module_path ';'
;
```

Where `module_path` is defined in the existing WESL grammar. 

The WGSL grammer is additionally extended as follows: 

```bnf
function_decl :
  attribute * 'patch' ? function_header compound_statement
;

extend global_decl : 
| include_decl
;
```

## Linking WESL Files

Linking a WESL file that supports both `include` and `patch` naturally introduces additional complexity when linking. Broadly speaking there are two approaches that could be taken to  support this feature. 

The first, more naïve approach, is to copy all the symbols from the module being included and simply replace the patched functions within the AST in the pass before
canonicalization. This may be more than somewhat inefficient however as there would likely be many duplicates included in output. This means you'd be relying heavily on dead code elimination, the wgsl compiler, and the underlying drivers to reduce register pressure. However this approach should be easy to ensure correctness with. 

The second is more forensic, requiring that a usage graph be created for all the patched functions in the module being included. The functions present within this graph would need to be copied into the including namespace, while functions not within the graph would need to resolve to the original module. 

What increases the complexity of both approaches is the support for module nesting. Inline modules are able to access the symbols of the outer namespaces. Both usage graph and symbol duplication approaches would therefore need to include these submodules in all calculations. An additional source of complexity is that an included module may itself use `include`. To reduce complexity, cycles should be strictly forbidden in compliant implementations. 

For the second approach, dead code elimination would additionally need to take extension usages into account. 

## Type Checking

Module type checking would need to be modified as follows: 
- additional symbols found within the included files would need to be considered in conjunction with the rest of the type checking rules.

## Drawbacks

Are there reasons as to why we should we not do this?

- This extension would introduce additional complexity in linking and type checking
- Allowing multiple includes suffers from some of the problems present in multiple inheritance. 
  - Forbidding clashing symbols mitigates some of these concerns
  - Module system in conjunction with this proposal would allow users to work around this
- Introduces additional function calls at extension points vs the templating approach. 
  - Whether this is a serious drawback given is TBD
  - Likely would require a concerted effort to benchmark performance cost
  - Future work could inline some of this code
- Does not address struct includes which may be a requirement for some use cases
  - Future proposals could extend this and behavioural semantics and implementations could learn from this proposal

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
  - Solves common challenge with PBR rendering
  - Allows for creation of smaller, more focused modules. 
    - This allows for better reuse and behavioural composition
  - Solves some use cases that would otherwise require generic modules
    - Not all however
    - Simpler to understand than generic modules
  - Patches are scoped to a namespace 
    - Better for language servers than monkey patching
  - Allows entrypoints to be injected into the global namespace 
  - Allows users to import symbols within a module - reducing verbosity without requiring an additional mechanism
- What other designs have been considered and what is the rationale for not choosing them?
  - See below
- What is the impact of not doing this?
  - Little extensibility in WESL specification without it
  - No means of including libraries into the global namespace preventing libraries from defining entry points
  - Would need an additional mechanism to make individual symbols available to the current namespace


### String Templating
Allows for arbitrary injection and construction of code at the specified points meaning that it is more expressive in some ways than the proposal above. It's fast. Cannot be easily statically analysed. Poor experience when considering language servers

### Single Inheritance
Con is it places an additional limitation on architecture. Pro is it avoids some issues seen multiple inheritance. 
Con is it would prevent usage of `include` as a means of introducing symbols into the current scope. 

## Future Work
 - Extension of structs
 - Structured inclusion of code fragments into method bodies?
