# ![wgsl-import-spec](branding.svg)

# Summary

We propose adding an importing mechanism to the WGSL shading language as an extension.

# Motivation

When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders and.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, nor does it provide adjacent features like namespaces, and there is no standardized extension yet.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume.

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. Using WGSL-with-imports both in Rust projects and in web projects should be possible.

## Guide-level explanation

This variant is semantically most similar to the Typescript syntax, but comes with a recursive grammar that makes importing multiple importable items less verbose. The grammar is also heavily inspired by [Gleam](https://gleam.run/).

By placing an import at the very top of a file, one can either import an entire module module, or only specific importable items, such as functions, structs or types.

```
// Importing a single item using a relative path
import ./lighting/pbr;

// Importing multiple items
import my/geom/sphere/{ draw, default_radius as foobar };

// Imports a whole module. Use it with `bevy_ui.name`
import bevy_ui/*;
```

These can then be used anywhere in the source code.

```
fn main() {
    bevy_ui.quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Both `bevy_ui` and `my` are packages in the current project. Language servers and related tools can look in a `wgsl.toml` file to find the location of the packages. This lets libraries be published to package managers, and users can import them with a simple syntax.

The first part is the file path, which is assumed to have a either `.wgsl` file extension,
or the extension we use for extended wgsl syntax (possibly `.wesl`).

Recursive import definitions are also supported, which leads to short and succinct import statements.

```
import bevy_pbr/{
  forward_io/VertexOutput,
  pbr_types/{PbrInput, pbr_input_new},
  pbr_bindings/*
};

fn main() {

}
```

## `wgsl.toml` file

The `wgsl.toml` file is a configuration file that tells the language server where to find the packages. It is similar to a `Cargo.toml` file in that regard.

```toml
name = "my"
edition = "2024"

[dependencies]
bevy_ui = { cargo = "bevy_ui" }
shader_wiz = { path = "./node_modules/shader_wiz/src/main.wgsl" }
```

We specify how to resolve the paths of packages instead of scanning folders for `*.wgsl` files.
In the Javascript world, it is common to have a `node_modules` folder with 10k files, which is not practical for a language server to scan.

We are planning on taking advantage of existing package managers, such as `cargo` for Rust, and `npm` for Javascript. This makes it easier for users to consume shaders, and makes sense for ecosystem-specific tools.

# Reference-level explanation

Imports must appear as the first items in a WGSL file. They import "importable items" (see [GLOSSARY.md](./GLOSSARY.md)).

An import statement is parsed as follows, with spaces and comments allowed between tokens:

```
main:
| 'import' import_path ';'

import_path:
| ('.' | '..') '/' import_path
| ident '/' (import_path | import_collection | item_import | star_import)

star_import:
| '*' ('as' ident)?

item_import:
| ident ('as' ident)?

import_collection:
| '{' (import_path | item_import) (',' (import_path | item_import))* ','? '}'
```

TODO: Should we restrict dots to only appear at the beginning of the path? It wouldn't restrict the user, but would make the grammar simpler.

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

```
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

```
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

```
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

## Preserved items

These items are preserved when importing a module. They are not imported, but will land in the final module with a mangled name.

- [Entry points](https://www.w3.org/TR/WGSL/#entry-points)
- [Pipeline overridable constants](https://www.w3.org/TR/WGSL/#override-decls)

## Name mangling

To join multiple modules into one, we need to make sure that names do not collide. This is done by name mangling.

Compiled WGSL files have a few exposed parts, namely the names of the entry functions and pipeline overridable constants. These need to be stable and predictable, so we will introduce a stable, predictable and human-readable name mangling scheme.

Finally, only importable items need to be mangled.

For name mangling, we introduce the concept of a **absolute module identifier**. This is an ID that is relative to the nearest _package_, and is unique for each module. For example, given a file `my/geom/sphere.wgsl`, and a package `my`, then the absolute module identifier is `my_geom_sphere`. It is always this, regardless of how the file is imported. If a file or folder name already contains an underscore, it is replaced by two underscores.

The name mangling scheme is as follows:

1. Underscores are used as separators. If a name already contains an underscore, it is replaced by two underscores.
2. Prefix each name with the absolute module identifier, followed by a single underscore.

For example, given a file `my/geom/sphere.wgsl` with a function `draw_now`, the mangled name would be `my_geom_sphere_draw__now`.

### Unmangling

We keep track of a stack of parts, and always add characters to the last part. It starts with a single empty part.

To unmangle a name, one goes over a mangled name from left to right.  
If two underscores are encountered, we add one underscore to the current part, and skip the second underscore.  
If only one underscore is encountered, we start a new part.
If anything else is encountered, we add it to the current part.

At the end, the final part is the item name, and the other parts are the module path.

## Identifier Resolution

The steps of identifier resolution are as follows:

1. Parse the import statements. Resolve the paths to the concrete file paths.
2. Bring the imported items into scope. All other items that the imported items depend on are also imported, _but not user-accessible in the current scope_.
   For example when importing `Foo` from `struct Foo { x: Bar; }`, we would import `Bar` as well. If the user types `Bar` in the source code, then that is an error.
3. Parse the WGSL.

For example, given two modules

```
// lighting.wgsl
struct Light {
    color: vec3;
}
struct LightSources {
    lights: array<Light>;
}
```

```
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

The downside is using additional tooling, and dealing with an additional translation layer. An additonal translation layer could lock shader authors out of certain WGSL features.

## Composing shader code at runtime

One alternative is to compose shader code at runtime, for example by joining together strings with WGSL code.
This has the major downside of not being statically analyzable. The IDE cannot provide autocompletion, and the compiler cannot check for errors.

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

# Unresolved questions

- Is .toml the best file format for the `wgsl.toml`? Some alternatives would be JSON/JSON5 and StrictYAML.

# Implementation

We are slowly working on adopting and adjusting the specification.
This work is happening in multiple tools, including [a WGSL language server](https://github.com/dannymcgee/vscode-wgsl/), a [Typescript WGSL linker](https://github.com/mighdoll/wgsl-linker) and [Bevy's WGSL-imports library](https://github.com/bevyengine/naga_oil).

# Test cases

We encourage share test cases for this proposal. This is a work in progress right now!

# Future possibilities

## Export

One natural extension of this proposal is to add explicit exports.
For one, this would allow library authors to hide certain functions or structs from the outside world. It would also enable re-exports, where a library re-exports a function from another library.

However, this adds some parser complexity, and is not necessary for the initial version of this proposal.

There are two variations of exports, which could be combined like in Typescript

### Exporting a list of items

A standalone export statement simply specifies which globals are exported.
Then, imports can be checked against the list of exports. This is very easy to parse and implement.

```
export { foo, bar };
```

And when one wants to export everything defined in the current file, they can use the `*` syntax.

```
export *;
```

To re-export an item, one can use the same syntax.

```
import my::lighting::{ pbr };

export { pbr };
```

### Exporting as an attribute

Exports can also be added as an attribute to the item itself.

```
@export
struct Foo {
    x: f32;
}

@export
fn foo() {}
```

This is more user friendly, but also more complex to parse. It requires a partial parsing of the WGSL file to find the exports and their names.
A future export specification would include the minimal WGSL syntax that is necessary to implement this.

## Namespaces

We hope that namespaces will be added to WGSL itself. Then, the importing mechanism can be extended to fully support namespaces, for example by treating each file as introducing its own namespace.

## Documentation comments

This proposal works nicely with documentation comments in WGSL. This would allow library authors to document their shaders, which would be very useful for consumers.

## Source maps

We encourage tooling authors to also implement source maps when implementing imports.

## Preprocessor

How a preprocessor would interact with this proposal is an open question for a future proposal.
