# Summary

We propose adding an importing mechanism to the WGSL shading language as an extension.

# Motivation

When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders and.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, nor does it provide adjacent features like namespaces, and there is no standardized extension yet.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume.

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. Using WGSL-with-imports both in Rust projects and in web projects should be possible.

## Guide-level explanation

The `import` statement extension brings items or entire modules into scope. Import statements map to files with minimal searching.

```wgsl
// Importing a single function using a relative path
import self::lighting::pbr;

// Importing multiple items
import my::geom::sphere::{ draw, default_radius as foobar };

// Imports a whole module. Use it with `bevy_ui::name`
import bevy_ui;
```

These can then be used anywhere in the source code.

```wgsl
fn main() {
    bevy_ui::quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Both `bevy_ui` and `my` are packages in the current project. Language servers and related tools can look in a `wgsl.toml` file to find the location of the packages. This lets libraries be published to package managers, and users can import them with a simple syntax.

Recursive import definitions are also supported, which leads to shorter import statements.

```wgsl
import bevy_pbr::{
  forward_io::VertexOutput,
  pbr_types::{PbrInput, pbr_input_new},
  pbr_bindings
};

fn main() {

}
```

To find the relevant items, the following algorithm is used:

Proceding left to right through the path segments, consider the segments `prev` and `seg`.
1. if `prev.wesl` exists and includes WESL elements, check if `seg` is one of those elements, e.g. `fn seg` or `namespace seg`.
1. else if the directory `prev/` exists, check to see if the file `seg.wesl` or the directory `seg/` is in the `prev/` directory;
1. error if a `seg` is not found. 

# Reference-level explanation

A WESL program is composed of a tree of WESL modules.

Imports must appear as the first items in a WESL file. They can import entire modules or individual  "importable items" (see [GLOSSARY.md](./GLOSSARY.md)).

An import statement is parsed as follows, with spaces and comments allowed between tokens:

```ebnf
translation_unit:
| import_statement* global_directive* global_decl* 

import_statement:  
| 'import' import_relative? import_path ';'  

import_relative:
| ('self' | 'crate' | 'super') '::' ('super' '::')*

import_path:
| (ident '::')+ (import_collection | item_import)  

item_import:
| ident ('as' ident)?

import_collection:
| '{' (import_path | item_import) (',' (import_path | item_import))* ','? '}'
```

Where `translation_unit` and `ident` are defined in the WGSL grammar.


An item import imports a single item. The item can be renamed with the `as` keyword.

An import collection imports multiple items, and allows for nested imports.

To resolve the import, the recursive structure is flattened out. Then, one iterates over each segment, and looks it up one by one.

1. We start with the first segment. 
    - `self` refers to the current module.
    - `super` refers to the parent module.
    - `crate` refers to the top level module of the current package.
    - `ident` must be a known package, usually found in the `wgsl.toml` file. It refers to the top level module of that package.
2. We take that as the "current module".
3. We repeatedly look at the next segment.
    1. Item in current module: Take that item. We must be at the last segment, otherwise it's an error.
    2. (Else if re-exported or inline module in current module: We continue with that module.)
    3. Else go to `current module path/ident.wesl`
       - File found: We take that file as the current module.
       - File not found: We assume an empty module as the current module, and continue with that.
       - (Re-exporting changes the path.)
       - (Inline modules do not have a path.)

For example

```wgsl
import bevy_pbr::forward_io::VertexOutput;
```
first looks for `bevy_pbr.wesl`.
`bevy_pbr.wesl` is found, and doesn't contain an item named `forward_io`.
Thus, we go to `bevy_pbr/forward_io.wesl`. It contains a struct named `VertexOutput`.

## Parsing module::importable_item in the source code

The syntax can also be used inline. To do so, we extend 
the [WGSL grammar](https://www.w3.org/TR/WGSL/#grammar-recursive-descent) as follows.

We introduce 

```ebnf
full_ident:
| import_relative? ident ('::' ident)*
```

and then replace `ident` with `full_ident` in the following places:

```ebnf
core_lhs_expression:
| full_ident
| ...

for_init: 
| full_ident func_call_statement.post.ident
| ...

for_update:
| full_ident func_call_statement.post.ident
| ...

global_decl: 
| attribute* 'fn' ident '(' ( attribute* full_ident ...
| ...
| 'alias' ident '=' full_ident ...
| ...

primary_expression: 
| full_ident template_elaborated_ident.post.ident
| full_ident template_elaborated_ident.post.ident argument_expression_list
| ...

statement: 
| ...
| full_ident template_elaborated_ident.post.ident argument_expression_list ';'
| ...

type_specifier:
| full_ident ...
```

TODO: Those sections need updating

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

These items are preserved when importing a module. Their name must be preserved. They will land in the final module, if they are being referenced.

- [Entry points](https://www.w3.org/TR/WGSL/#entry-points)
- [Pipeline overridable constants](https://www.w3.org/TR/WGSL/#override-decls)


## `const_assert`

Generally, WGSL elements are included if they are recursively referenced from the root module (use analysis). But `const_assert` statements are also included if they are in the same module or namespace as a referenced element.

-   ```wgsl
    // main.wgsl:
    import  foo/bar;
    fn main() { bar(); }

    // foo.wgsl:
    import ./zig;
    const_assert(1 > 0); // included in link because bar is used
    fn bar() { }
    fn miz() { zig.zag() }

    // zig.wgsl:
    const_assert(2 < 0); // not included in link
    fn zag() { }
    ```


## Identifier Resolution

The steps of identifier resolution are as follows:

1. Parse the import statements. Resolve the paths to the concrete file paths.
2. Bring the imported items into scope. All other items that the imported items depend on are also imported, _but not user-accessible in the current scope_.
   For example when importing `Foo` from `struct Foo { x: Bar; }`, we would import `Bar` as well. If the user types `Bar` in the source code, then that is an error.
3. Parse the WGSL.

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
