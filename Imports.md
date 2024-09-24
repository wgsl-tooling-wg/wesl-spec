# ![wgsl-import-spec](branding.svg)

# Summary

We propose adding an importing mechanism to the WGSL shading language as an extension.

## TBD

* Require braces for wgsl items? e.g. `import ./lighting/{ pbr };`
  * If we require braces for wgsl items, could do `import ./foo/bar` instead of `import ./foo/bar/*`?
* Add example for recursive imports

# Motivation

When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders and.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, nor does it provide adjacent features like namespaces, and there is no standardized extension yet.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume.

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. Using WGSL-with-imports both in Rust projects and in web projects should be possible.

## Guide-level explanation

The `import` statement extension is designed to appear somewhat familiar to TypeScript syntax,
but with a Rust like recursive grammar to make importing
multiple importable items less verbose.
The syntax is also heavily inspired by [Gleam](https://gleam.run/).

By placing an import at the very top of a file, one can either import an entire module, or only specific importable items, such as functions, structs or types.

```wgsl
// Importing a single item using a relative path
import ./lighting/pbr;

// Importing multiple items
import my/geom/sphere/{ draw, default_radius as foobar };

// Imports a whole module. Use it with `bevy_ui.name`
import bevy_ui/*;
```

These can then be used anywhere in the source code.

```wgsl
fn main() {
    bevy_ui.quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Both `bevy_ui` and `my` are packages in the current project. Language servers and related tools can look in a `wgsl.toml` file to find the location of the packages. This lets libraries be published to package managers, and users can import them with a simple syntax.

The first part is the file path, which is assumed to have a either `.wgsl` file extension,
or the extension we use for extended wgsl syntax (possibly `.wesl`).

Recursive import definitions are also supported, which leads to short and succinct import statements.

```wgsl
import bevy_pbr/{
  forward_io/VertexOutput,
  pbr_types/{PbrInput, pbr_input_new},
  pbr_bindings/*
};

fn main() {

}
```

# Reference-level explanation

Imports must appear as the first items in a WGSL file. They import "importable items" (see [GLOSSARY.md](./GLOSSARY.md)).

An import statement is parsed as follows, with spaces and comments allowed between tokens:

```wgsl
main:  
| 'import' import_relative? import_path ';'  

import_relative:  
| ('.' | '..') '/' ('..' '/')*  

import_path:  
| ident '/' (import_path | import_collection | item_import | star_import)  

star_import:
| '*' ('as' ident)?

item_import:
| ident ('as' ident)?

import_collection:
| '{' (import_path | item_import) (',' (import_path | item_import))* ','? '}'
```

Where `ident` is defined in the WGSL grammar.

An import consists of

- A path part, which either is a relative path ('.' and '..'), or points at a known package (ident).

  - Nested path segments are joined.
  - Everything before the final slash is part of the path.
  - The final part is the file name.

- Items to import.

A star import behaves like a star import in Typescript, except that the name can be inferred from the file name. **Notably**, it does *not* import all the items individually. Instead, it groups them all under a single name!

An item import imports a single item. The item can be renamed with the `as` keyword.

An import collection imports multiple items, and allows for nested imports.

## Examples

To compare it to the more widely known Typescript syntax, here are some examples.

<table>
<thead>
<tr>
<th>WGSL</th>
<th>Typescript</th>
</tr>
</thead>
<tbody>
<tr>
<td>

```wgsl
import ../geom/sphere/{draw, default_radius as foobar};
```

</td>
<td>

```ts
import { draw, default_radius as foobar } from '../geom/sphere.wgsl';
```

</td>
</tr>
<tr>
<td>

```wgsl
import bevy_ui/*;
```

</td>
<td>

```ts
import * as bevy_ui from 'bevy_ui.wgsl';
```

</td>
</tr>
<tr>
<td>

```wgsl
import bevy_pbr/{ 
    forward_io/VertexOutput, 
    pbr_types/{PbrInput, pbr_input_new}, 
    pbr_bindings/* as pbr_b
};
```

</td>
<td>

```ts
import { VertexOutput } from 'bevy_pbr/forward_io.wgsl';
import { PbrInput, pbr_input_new } from 'bevy_pbr/pbr_types.wgsl';
import * as pbr_b from 'bevy_pbr/pbr_bindings.wgsl';
```

</td>
</tr>
</tbody>
</table>

## Parsing module.importable_item in the source code

For tools that are parsing WGSL,
here's how to extend the
[WGSL grammar](https://www.w3.org/TR/WGSL/#grammar-recursive-descent)
to parse imports:

About extending the grammar:
This one gets an additional meaning. A module property can also be a component.
component_or_swizzle_specifier:
| '.' member_ident component_or_swizzle_specifier ?

This needs to be extended to support ident.ident
for_init:
| ident.ident func_call_statement.post.ident

for_update:
| ident.ident func_call_statement.post.ident

global_decl:
| attribute _ 'fn' ident ... '->' attribute _ ident.ident
| 'alias' ident '=' ident.ident template_elaborated_ident.post.ident ';'

primary_expression:
| ident.ident template_elaborated_ident.post.ident
| ident.ident template_elaborated_ident.post.ident argument_expression_list

statement:
| ident.ident template_elaborated_ident.post.ident argument_expression_list ';'

type_specifier:
| ident.ident ...

## Behaviour-changing items

These items cannot be imported, but they affect module behavior:

- [Extensions](https://www.w3.org/TR/WGSL/#extensions)
- [Global diagnostic filters](https://www.w3.org/TR/WGSL/#global-diagnostic-directive)

These are always set at the very top in the main module, and affect all imported modules. They come before the imports.

When, during parsing of imported modules, we encounter an extension or a global diagnostic filter, we check if the main module enables it, or sets the filter.
If yes, everything is fine. If not, we throw an error.

(In naga-oil entrypoints are lowered to normal functions.
Not clear we should preserve that, it's against the spec,
but noting current behaviour that is being used in the wild.)

## Preserved items

These items are preserved when importing a module. They are not imported, but will land in the final module with a mangled name.

- [Entry points](https://www.w3.org/TR/WGSL/#entry-points)
- [Pipeline overridable constants](https://www.w3.org/TR/WGSL/#override-decls)

## Identifier Resolution

The steps of identifier resolution are as follows:

1. Parse the import statements. Resolve the paths to the concrete file paths.
2. Bring the imported items into scope. All other items that the imported items depend on are also imported, _but not user-accessible in the current scope_.
   For example when importing `Foo` from `struct Foo { x: Bar; }`, we would import `Bar` as well. If the user types `Bar` in the source code, then that is an error.
3. Parse the WGSL.

For example, given two modules

```wgsl
// lighting.wgsl
struct Light {
    color: vec3;
}
struct LightSources {
    lights: array<Light>;
}
```

```wgsl
// main.wgsl
import lighting::{ LightSources };

// LightSources can be used
fn clear_lights(lights: LightSources) -> LightSources {
    for (var i = 0; i < lights.lights.length(); i = i + 1) {
        // We can access everything inside of LightSources
        let light = lights.lights[i];

        // However, the user cannot use the type "Light"
        // let light: Light <== error, Light is not imported

        light.color = vec3(0.0, 0.0, 0.0);
    }
    return lights;
}
```

## Producing the final module

The final module is produced by taking each imported module (in topological order, with the main module last), resolving and mangling all the globals, and joining the resulting pieces of code together.

All entry points and pipeline overridable constants from the imported modules are also mangled and land in the output.

Dead code elimination is allowed, but not required.

# Drawbacks

Are there reasons as to why we should we not do this?

- This introduces yet another importing syntax that developers have to learn, instead of using a standard syntax.
- To implement the name mangling, one has to parse WGSL code! This is not trivial, and requires a partial WGSL parser.
- Paths in import statements must consist of valid WGSL identifiers, which can be limiting. This limitation could be lifted by allowing arbitrary strings in import paths, but would make the implementation more complex.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

## Not agreeing on a standard

One major upside of standardizing it is that it becomes practical for language servers to support it.

The usual alternative is that one library, like shaderc, becomes very popular and the standard ends up being "whatever popular library XYZ does".

An open process lets us find a better solution.

## Preprocessor `#include <lighting.wgsl>`

One alternative, which is common in the GLSL and C worlds, is an including mechanism which simply copy-pastes existing code. A major upside is that this is very simple to implement.

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

Another drawback is that using the same name twice is impossible. In C-land, this leads to pseudo-namespaces, where major libraries will prefix all of their functions with a few symbols. An example of this is the Vulkan API `vkBeginCommandBuffer` or `vkCmdDraw`.

A future drawback is that "privacy" or "visibility" becomes very difficult to implement. Everything that is imported is automatically public and easily accessible.
In C-land, the workaround is using header files. In other languages, such as Python, the convention ends up being "anything prefixed with an underscore `_` is private".

## Typescript-like imports

TODO: Main reason is just that they're more verbose

## Rust-like imports

TODO: Needs something akin to a `mod` statement, otherwise ambiguity.

## Putting exports in comments

This would have the advantage of letting some existing WGSL tools ignore the new syntax. For example, a WGSL formatter would not need to know about imports, and could just format the code as usual.

## Using an alternative shader language

There are multiple higher level shading languages, such as [slang](https://github.com/shader-slang/slang) or [Rust-GPU](https://github.com/EmbarkStudios/rust-gpu) which support imports. They also support more features that WGSL currently does not offer. For complex projects, this can very much pay off.

The downside is using additional tooling, and dealing with an additional translation layer.
An additonal translation layer could lock shader authors out of certain WGSL features.

Also, higher level GPU languages are typically processed at build time,
which precludes using language features to adapt to runtime conditions
like GPU characteristics or user settings.

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

## Namespaces

We hope that namespaces will be added to WGSL itself. Then, the importing mechanism can be extended to fully support namespaces, for example by treating each file as introducing its own namespace.

## Documentation comments

This proposal works nicely with documentation comments in WGSL. This would allow library authors to document their shaders, which would be very useful for consumers.

## Source maps

We encourage tooling authors to also implement source maps when implementing imports.

## Preprocessor

How a preprocessor would interact with this proposal is an open question for a future proposal.
See [Conditional Compilation](./ConditionalCompilation.md).

## Scoped imports

Allow imports that are only active within one function?
