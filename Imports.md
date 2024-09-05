# ![wgsl-import-spec](branding.svg)

# Summary

We propose adding an importing mechanism to the WGSL shading language as an extension.

# Motivation

When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders and.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, nor does it provide adjacent features like namespaces, and there is no standardized extension yet.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume.

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. Using WGSL-with-imports both in Rust projects and in web projects should be possible.

# Guide-level explanation

The `load` statement extension is designed to appear somewhat familiar to Ruby `require` syntax, and specifies that a given file should be loaded by the linker and all of its symbols be made available to the global context. 

A given file is loaded only once and it is also legal for files to cyclically refer to one another. 

One important restriction of this proposal is that it is illegal for symbols to have the same name, unless as per the [Extends](./Extends.md) proposal a symbol is declared virtual. 

All loaded entrypoints in the global scope are guaranteed to be available to the WebGPU API. However this guarantee does not extend to bindings as these may be removed by dead code elimination if not used in any entry point. 

An example of how a shader may use loads is as follows:

In a file called `utils.wesl`:

```wgsl 
@binding(0) @group(0) var<uniform> frame : u32;

fn current_frame_plus_two() -> f32 {
  return frame + 2.0;
}
```

In a file called `entrypoint.wesl` in the same directory as `utils.wesl`:

```wgsl
load ./utils;

@fragment
fn frag_main() -> @location(0) vec4f {
  return vec4(1, sin(current_frame_plus_two()), 0, 1);
}
```

Note that all names in the above example are preserved in linker output.


# Reference-level explanation

A load statement is parsed as follows, with spaces and comments allowed between tokens:

```
main:  
| 'load' load_relative? load_path ';'  
;

load_relative:  
| ('.' | '..') '/' ('..' '/')*  
| '/'
;

load_path:  
| ident ('/' ident)*
;
```

Where `ident` is defined in the WGSL grammar.

A load consists of

- A path part, which either is a relative path ('.', '..' or '/'), or points at a known package (ident).
  - Nested path segments are joined.
  - Everything before the final slash is part of the path.
  - The final part is the file name.

## Items to import.

This proposal brings all items from the requested file recursively into scope. 
This means that if file a loads `abc.wesl` which uses `def.wesl`, then the symbols from 
both `abc.wesl` _and_ `def.wesl` will be in scope. Note that both `wgsl` and `wesl` files may be 
loaded, so it is a link time error to have both a wgsl file and a wesl file in the same directory with 
the same name.

If two loaded files have any clashing global symbols, then implementions of this specification are expected to produce an error.

Entrypoints in the global scope that are loaded either directly or indirectly via the main file are considered used. 

Typical linker implementations would likely want to analyse usages from these entrypoints to eliminate unused globals and bindings.

## Examples

## Related Specifications

This proposal on its own is impractical for developing a robust ecosystem due to lack of namespacing. The [Modules](./Modules.md) proposal is likely necessary to make this import proposal workable.

## Behaviour-changing items

These items cannot be imported, but they affect module behavior:

- [Extensions](https://www.w3.org/TR/WGSL/#extensions)
- [Global diagnostic filters](https://www.w3.org/TR/WGSL/#global-diagnostic-directive)

These are always set at the very top in the main module, and affect all imported modules. They come before the imports.

When, during parsing of imported modules, we encounter an extension or a global diagnostic filter, we check if the main module enables it, or sets the filter.

If yes, everything is fine. If not, we throw an error.

## Producing the final module

The final module is produced by taking each loaded module in topological order, with the main module last, and ensuring that each module is only taken once. The results are then concatenated together.

Dead code elimination is allowed, but not required.

## Drawbacks

Are there reasons as to why we should we not do this?

- This introduces yet another importing syntax that developers have to learn, instead of using a standard syntax.
- Paths in load statements must consist of valid WGSL identifiers, which can be limiting. This limitation could be lifted by allowing arbitrary strings in import paths, but would make the implementation more complex.
- The design of this proposal requires that developers of reusable shader libraries shoulder the responsibility to prevent naming collisions. This would be ameliorated by the [Modules](./Modules.md) proposal.
- This essentially makes _all loads_ wildcard imports. This could make language servers and usage analysis harder to implement though one would hope the uniformity of this proposal would help somewhat in reducing this complexity.


# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

## Not agreeing on a standard

One major upside of standardizing it is that it becomes practical for language servers to support it.

The usual alternative is that one library, like shaderc, becomes very popular and the standard ends up being "whatever popular library XYZ does".

An open process lets us find a better solution.

# Simplicity

This model is simple to implement and understand, even for users not familiar with languages like typescript or rust. It also avoids the name mangling problem in the WebGPU visible API. 

## Preprocessor `#include <lighting.wgsl>`

One close alternative, which is common in the GLSL and C worlds, is an including mechanism which simply copy-pastes existing code. A major upside is that this is very simple to implement.

One drawback is that importing the same shader multiple times, which can also happen indirectly, does not work without other preprocessor features.

```c
// A.wgsl
#include <lighting.wgsl>
#include <math.wgsl>
```

```c
// lighting.wgsl
#include <math.wgsl>
```

would not work, since anything defined in `math.wgsl` would be imported twice. In C-land, this is solved by using _include guards_.

Another drawback is that using the same name twice is impossible. In C-land, this leads to pseudo-namespaces, where major libraries will prefix all of their functions with a few symbols. An example of this is the Vulkan API `vkBeginCommandBuffer` or `vkCmdDraw`. The [Modules](./Modules.md) proposal would avoid this pitfall.

A future drawback is that "privacy" or "visibility" becomes very difficult to implement. Everything that is imported is automatically public and easily accessible.

In C-land, the workaround is using header files. In other languages, such as Python, the convention ends up being "anything prefixed with an underscore `_` is private".

Visibility in our proposal in contrast would be via the [Module Interfaces](./ModulesInterfaces.md) proposal and 
would be optional, allowing regular wgsl files to be imported without modification.

## Typescript-like imports

TODO: Main reason is just that they're more verbose. Also is more complicated than this proposal

## Rust-like imports

TODO: Needs something akin to a `mod` statement, otherwise ambiguity. Also is more complicated than this proposal

## Putting exports in comments

This would have the advantage of letting some existing WGSL tools ignore the new syntax. For example, a WGSL formatter would not need to know about imports, and could just format the code as usual.

This could be considered in conjunction with or in addition to this proposal. 

## Using an alternative shader language

There are multiple higher level shading languages, such as [slang](https://github.com/shader-slang/slang) or [Rust-GPU](https://github.com/EmbarkStudios/rust-gpu) which support imports. They also support more features that WGSL currently does not offer. For complex projects, this can very much pay off.

The downside is using additional tooling, and dealing with an additional translation layer.
An additional translation layer could lock shader authors out of certain WGSL features.

Also, higher level GPU languages are typically processed at build time, which precludes using language features to adapt to runtime conditions like GPU characteristics or user settings.

## Composing shader code as strings at runtime

One alternative is to compose shader code at runtime
by simply joining together strings with WGSL code, perhaps
with some string templating for flexibility.
This has the major downside of not being statically analyzable.
The IDE cannot provide autocompletion,
and a language server cannot check for errors.

A linker that understands imports also typically
composes shader strings, and can link at runtime.
But a linker uses its more sophisticated understanding of WGSL
to drive composition.
For example, a linker can identify imports
that are needed by other imports,
automating shader composition for users.

<!--
# Future: Prior art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC. Please also take into consideration that rust sometimes intentionally diverges from common language features.
-->

# Implementation

We are slowly working on adopting and adjusting the specification.
This work is happening in multiple tools, including [a WGSL language server](https://github.com/dannymcgee/vscode-wgsl/), a [Typescript WGSL linker](https://github.com/mighdoll/wgsl-linker) and [Bevy's WGSL-imports library](https://github.com/bevyengine/naga_oil).

# Test cases

Test cases will be available on
[github](https://github.com/wgsl-tooling-wg/wgsl-import-spec).

# Future possibilities

## Documentation comments

This proposal works nicely with documentation comments in WGSL. This would allow library authors to document their shaders, which would be very useful for consumers.

## Source maps

We encourage tooling authors to also implement source maps when implementing imports.

## Preprocessor

How a preprocessor would interact with this proposal is an open question for a future proposal.
See [Conditional Compilation](./ConditionalCompilation.md).
