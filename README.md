
# Summary

We propose adding an importing mechanism to the WGSL shading language as an extension.

# Motivation

When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders and.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, and there is no standardized extension yet. 

One important aspect is getting buy-in from **WGSL language servers**, as shader development with IDEs should be well supported.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume. 

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. I personally am using using WGSL both in Rust projects, and in web projects. We should not make it ecosystem specific.

## Variant A - Guide-level explanation

This variant is based on Rust syntax.

One can either import a whole module, or only specific functions.

```
#import bevy_ui;
#import my::module::{func1, func2 as foobar};
```

TODO: Explain that we either need #export, or that we need the curly brackets around functions.

These can then be used

```
fn main() {
    bevy_ui::sphere(vec2f(0.0, 1.0), 1.0);
    let a = func1(3);
}
```

Both `bevy_ui` and `my` are packages in the current project. Tools can look in a `Wgsl.toml` file to find the location of the packages.
This lets libraries be published to package managers, and users can import them with a simple syntax.

Relative paths are also supported, so one can import a shader from a different directory.

```
#import super::my::module;
#import super::super::other::module;
```

The name `#import` could also be changed to `#use` or `use`, to match the Rust convention. We should also consider how it interacts with a preprocessor, for example `use` without a hashtag might imply that it's not being analyzed in the preprocessing stage.
Do note that a general preprocessor is not part of this proposal.


## Variant B - Guide-level explanation

This variant is based on Typescript syntax.

One can either import a whole module, or only specific functions.

```
import * as bevy_ui from "bevy_ui";
import { func1, func2 as foobar } from "my/module";
```

These can then be used

```
fn main() {
    bevy_ui::sphere(vec2f(0.0, 1.0), 1.0);
    let a = func1(3);
}
```

Relative paths are also supported, so one can import a shader from a different directory.

```
import { bar } from "../my/module";
```

## `Wgsl.toml` file

The `Wgsl.toml` file is a configuration file that tells the language server where to find the packages. It is similar to a `Cargo.toml` file in that regard.

```toml
edition = "2024"

[dependencies]
bevy_ui = { cargo = "bevy_ui" }
my = { path = "./relative/path" }
shader_wiz = { path = "./node_modules/shader_wiz/src/main.wgsl" }
``` 

We specify the paths instead of scanning folders for `*.wgsl` files. 
In the Javascript world, it is common to have a `node_modules` folder with 10k files, which is not practical for a language server to scan.

We also recommend supporting language specific package managers, such as `cargo` for Rust, and `npm` for Javascript. This makes it easier for users to consume shaders, and makes sense for ecosystem-specific tools.

TODO: Is .toml the best file format for this? Alternatives would be
- JSON (or JSON5?)
- YAML (well, StrictYAML)

TODO: How should the importing syntax work?
- Have re-exports or similar. I never want to be forced to write `module::submodule::anotherone::help::pls::my_func()`.


# TODO: Guide-level explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should think about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# TODO: Reference-level explanation

TODO: Explain name mangling
TODO: When I import foo, and it uses a type FooBar, then FooBar is implicitly imported as well. However, the user cannot type "FooBar" in the source code, because it hasn't been explicitly imported.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

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

# TODO: Unresolved questions

- Namespaces https://github.com/gpuweb/gpuweb/issues/777

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# TODO: Test cases

We should have a repository with lots of shared test cases for implementations.

# Future possibilities

## Visibility

One natural extension of this proposal is to add visibility modifiers to WGSL. This would allow users to hide certain functions or structs from the outside world, which would be very useful for library authors. The syntax for this could be
- `pub fn my_func() { ... }`: Copying Rust
- `@export struct MyStruct { ... }`: Syntax used by [usegpu](https://usegpu.live/docs/reference-library-@use-gpu-shader)
- `#export`: A preprocessor macro, [wgsl-linker](https://github.com/mighdoll/wgsl-linker) implements this syntax
- `fn<export> my_func() { ... }`: Using the WGSL convention of using angle brackets for modifiers

## Documentation comments

One natural extension of this proposal is to add documentation comments to WGSL. This would allow users to document their shaders, which would be very useful for library authors.

## Source maps

We encourage tooling authors to also implement source maps when implementing imports.

## Preprocessor

How a future preprocessor proposal would interact with this proposal is an open question.
