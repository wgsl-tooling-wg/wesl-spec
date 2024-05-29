
# Summary

We propose adding an importing mechanism to the WGSL shading language as an extension.

# Motivation

When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders and.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, nor does it provide adjacent features like namespaces, and there is no standardized extension yet. 

One important aspect is getting buy-in from **WGSL language servers**, as shader development with IDEs should be well supported.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume. 

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. Using WGSL-with-imports both in Rust projects and in web projects should be possible.

## Variant A - Guide-level explanation

This variant is based on Rust syntax.

By placing an import at the very top of a file, one can either import a whole module, or only specific items, such as functions, structs or types. To import an item, one needs to use curly brackets.

```
import bevy_ui;
import my::lighting::{ pbr };
import my::geom::sphere::{ draw, default_radius as foobar };
```

These can then be used anywhere in the source code.

```
fn main() {
    bevy_ui::quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Both `bevy_ui` and `my` are packages in the current project. Tools can look in a `wgsl.toml` file to find the location of the packages. This lets libraries be published to package managers, and users can import them with a simple syntax.

The parts that come after the package name are the path to the concrete file, and the file is assumed to have a `.wgsl` file extension.

Relative paths are also supported, so one can import a shader from a different directory.

```
import super::sphere;
import super::super::lighting::{ pbr };
```


## Variant B - Guide-level explanation

This variant is based on Typescript syntax.

By placing an import at the very top of a file, one can either import a whole module, or only specific items, such as functions, structs or types. Paths can either be relative, or start with a package name. They then continue with a path to the concrete file.

```
import * as bevy_ui from "bevy_ui";
import { pbr } from "my/lighting.wgsl";
import { draw, default_radius as foobar } from "my/geom/sphere.wgsl";
```

These can then be used

```
fn main() {
    bevy_ui::quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Tools can look in a `wgsl.toml` file to find the location of the packages.

Relative paths are also supported, so one can import a shader from a different directory.

```
import * as sphere from "./sphere.wgsl";
import { pbr } from "./../lighting.wgsl";
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

We specify the paths of packages instead of scanning folders for `*.wgsl` files. 
In the Javascript world, it is common to have a `node_modules` folder with 10k files, which is not practical for a language server to scan.

We also recommend supporting language specific package managers, such as `cargo` for Rust, and `npm` for Javascript. This makes it easier for users to consume shaders, and makes sense for ecosystem-specific tools.

# Reference-level explanation

Imports must appear as the first items in a WGSL file.  

## Variant A - Reference-level explanation

An import statement is parsed as follows, with spaces and comments allowed between tokens:

```
import:
| 'import' path

path:
| ident ('::' ident)* ('::' '{' item_list '}' )?

item_list:
| ident (',' ident)* (',')?
```

And `ident` is defined in the WGSL grammar. 

After parsing, the semantics is as follows:

1. If the path starts with `super`, it is a relative path. The number of `super` determines how many directories to go up, with one `super` corresponding to the current *directory*.  
If the path starts with a package name, it is an absolute path. The package name is looked up in the `wgsl.toml` file, and the path is appended to the package path.
2. The other identifiers are appended to the file path. The final one gets a `.wgsl` extension.
3. If the path has a `::{ ... }` part, the items listed in the curly brackets are imported.  
Otherwise, the module is imported and can be used in code like `sphere::draw()`.

As an example, `import super::sphere` would be

1. `super` means current directory.
2. `sphere` is the file name. The final path is `./sphere.wgsl`, and is relative to the current wgsl file.

And `import my::lighting::{ pbr }` would be
1. `my` is a package name. The path is looked up in the `wgsl.toml` file. With the `wgsl.toml` above, it is the name of our current project. If the project lives at `/home/username/my-cute-project`, then that is the path.
2. `lighting` is the file name. The final path is `/home/username/my-cute-project/lighting.wgsl`.


<details>
  <summary>Why do import items need curly brackets?</summary>
  
The curly brackets are necessary to distinguish between importing a whole module and importing only specific items. Otherwise `import my::lighting` would be ambiguous. If we attempt to heuristically resolve this, it can lead to confusing situations where the semantics suddenly changes. For example, if the user is commenting out a function, then the import statement could switch between importing a single function and importing the whole module. This is not a good user experience.
</details>

## Variant B - Reference-level explanation

An import statement is parsed as follows, with spaces and comments allowed between tokens:

```
import:
| 'import' item_list path

path:
| string

item_list:
| '*' 'as' ident
| '{' ident_list '}'

ident_list:
| ident (',' ident)* (',')?
```

Where `ident` is defined in the WGSL grammar.


**Unresolved question: What exactly is a file path?**

The item list is a list of items that are imported. If the item list is empty, the whole module is imported.

`path` is a quoted string that resolves to a file path. If it starts with `.` or `..`, it is a relative path. If it starts with a package name, it gets resolved to the package path. It may not start with anything else, such as `/` or `C:` or `http://`.

## Module names in code

When the user imports a module, they can use the module name in the code. For example, if they import `my::lighting`, they can use `lighting::pbr` in the code.

Since double colons are not allowed in the WGSL grammar, this is a safe way to introduce them without having to introduce a new type of WGSL parser.

## Importable items

**Which items can be imported?**
What exactly can be imported?

- Structs: Yes
- Functions: Yes 
- [Extensions](https://www.w3.org/TR/WGSL/#extensions): ??
- [Global diagnostic filters](https://www.w3.org/TR/WGSL/#global-diagnostic-directive): ???
- Type aliases: Yes
- [Const declarations, override declarations](https://www.w3.org/TR/WGSL/#value-decls): Yes
- [Var declarations](https://www.w3.org/TR/WGSL/#var-decls): Probably yes 
  - Importing a `var<private> foo: f32;` looks odd
  - Bindings might not compose nicely, because they have a fixed index `@group(0) @binding(2) var<uniform> param: Params;`


## Name mangling

To join multiple modules into one, we need to make sure that names do not collide. This is done by name mangling.

Compiled WGSL files have a few exposed parts, namely the names of the entry functions and pipeline overridable constants. These need to be stable and predictable, so we will introduce a stable, predictable and human-readable name mangling scheme.

Finally, only importable items need to be mangled. 

For name mangling, we introduce the concept of a **absolute module identifier**. This is an ID that is relative to the root of the project, and is unique for each module. For example `my::lighting` would be `my_lighting`. Relative paths are resolved to absolute module identifiers. Underscores are used as separators. If a file or folder name already contains an underscore, it is replaced by two underscores.


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

## Implementation Semantics

One way of implementing this with the desired semantics is as follows:

TODO:
<!--

When I import foo, and it uses a type FooBar, then FooBar is implicitly imported as well. However, the user cannot type "FooBar" in the source code, because it hasn't been explicitly imported.

alias vec2f = u32; is probably valid WGSL code, and can very much screw up code that comes after it.

1. Replace mdule names in code with ???

1. Parse the import statements.
2. Parse the WGSL code to a syntax tree. This can be done with an existing parser.

1. Parse the import statements.
2. Parse the WGSL to a syntax tree.
3. Mangle the names ???
3. Add the imported items without mangling to the syntax tree. Every type that is referenced by the imports is also imported with a random UUID name.
4. 
    
Unnamable names
-->
# TODO: Drawbacks

Why should we not do this?

# TODO: Rationale and alternatives

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
would not work, since anything defined in `math.wgsl` would be imported twice. In C-land, this is solved by using *include guards*.


Another drawback is that using the same name twice is impossible. In C-land, this leads to pseudo-namespaces, where major libraries will prefix all of their functions with a few symbols. An example of this is the Vulkan API `vkBeginCommandBuffer` or `vkCmdDraw`.

A future drawback is that "privacy" or "visibility" becomes very difficult to implement. Everything that is imported is automatically public and easily accessible.
In C-land, the workaround is using header files. In other languages, such as Python, the convention ends up being "anything prefixed with an underscore `_` is private".

## Using a higher level language

There are multiple higher level shading languages, such as slang (TODO: link) or Rust-GPU (TODO: link) which support imports. They also support more features that WGSL currently does not offer. For complex projects, this can very much pay off.

The downside is using additional tooling, and dealing with an additional translation layer. An additonal translation layer could lock shader authors out of certain WGSL features.

This is always a trade-off, and I believe that offering multiple solid choices is the ideal option.

# TODO: Prior art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC. Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions

- Is .toml the best file format for the `wgsl.toml`? Some alternatives would be JSON/JSON5 and StrictYAML.


- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Test cases

We should add shared test cases for implementations to this repository.

# Future possibilities

## Export

One natural extension of this proposal is to add explicit exports.
For one, this would allow library authors to hide certain functions or structs from the outside world. It would also enable re-exports, where a library re-exports a function from another library.

However, this adds a significant amount of parser complexity, and is not necessary for the initial version of this proposal.

## Namespaces

We hope that namespaces will be added to WGSL itself. Then, the importing mechanism can be extended to fully support namespaces, for example by treating each file as introducing its own namespace.

## Documentation comments

This proposal works nicely with documentation comments in WGSL. This would allow library authors to document their shaders, which would be very useful for consumers.

## Source maps

We encourage tooling authors to also implement source maps when implementing imports.

## Preprocessor

How a preprocessor would interact with this proposal is an open question for a future proposal.
