# A Community Standard for Enhanced WGSL

We propose building an enhanced version of WebGPU's
[WGSL](https://www.w3.org/TR/WGSL/) shading language
using community tooling.
We want to see WGSL enhancements shared more broadly.
We want to enable a WGSL library ecosystem.

Many community projects have independently uncovered
a need for WGSL enhancements like module composition,
conditional compilation,
and simple templating.
Fancier features like virtual functions and generics
have also proven valuable.
By evolving a shared community set of WGSL enhancement definitions,
we can build common tools.
Better tooling benefits everyone,
including smaller projects that can't afford to build
their own enhancements and associated tools.

We'd like to
enable authors to publish resusable WGSL libraries
on npm and crates.io.
Sharing of WGSL code is limited today,
partly because users need to manually rewrite copy-pasted code fragments.
By combining a standardized and versioned set of extensions
with some packaging standards, we can
make it easy to publish and automatic to use WGSL libraries.
A standard community library format is key to enable an WGSL library ecosystem.

Join us on [github](https://github.com/wgsl-tooling-wg/wgsl-import-spec)
or [discord](https://discord.gg/FXhZDV8V) to help.

## Goals

Our ambition is to create a modestly improved, practical variant of WGSL.
The result should feel like WGSL with a few things added.

## Enhanced WGSL Programmers

Enhanced WGSL programmers are very likely to write shader programs
only part time.
And our programmers will learn WGSL first before learning our extensions.
Our enhancements should be easy for our target programmers to learn
and to remember.

We imagine enhanced WGSL programmers will have learned WGSL,
but not be familiar with other shader languages.
They're very likely familiar with JavaScript
and probably TypeScript. A smaller but significant portion of our
target programmers will know Rust
and Rust users are likely early adopters.
Some Python, Java, C++ familiarity is likely,
but we'll imagine that our target programmers took a class or two
and don't use those languages as often.

The target project size for enhanced WGSL ranges from tiny programs
up to teams of few dozen programmers.
Software engineering features useful for very large teams or very large
projects are unlikely to be needed for enhanced WGSL.

## WGSL Enhancements

We're aiming to prioritize WGSL enhancements that are: 1) important for community projects,
2) feel natural to the WGSL programmer,
and 3) reasonable to integrate into tools.

The simple enhancements we want to find will take some time to stabilize.
Adventerous projects will be able to opt in to experimental enhancements
to try alongside stable features.

A WGSL enhancement for `import` statements
to enable importing WGSL elements like functions from other files is our first priority to
stabilize. See spec draft here: **[imports](./Imports.md)**.

Several other WGSL enhancements
are on the [roadmap](#wgsl-enhancements-roadmap).

## Tools

We'll build and promote tools to support community standard WGSL.

**Linking** - Tools to bundle wgsl files together
while translating extended WGSL into vanilla W3C WGSL.

**Test Cases** - A collection of examples to help tools interoperate.

**WGSL Language Servers** - Enable syntax directed editing in IDEs.
Shader development with IDEs should be well supported.

**Syntax Highlighters** - Display pretty extended WGSL on web pages with
tools like Shiki and CodeMirror.

**Extended Grammars** - Grammars for TextMate, Kate, tree-sitter, etc. will enable support from a broad array of editors and highlighters.

**More Tools** - We'd like to see documentation extractors (akin to rustdoc/javadoc),
prettifiers, other editors (e.g. Kate), TextMate, etc. that understand the extended version of WGSL.

## Stability

As a community project,
we can iterate quickly on experimental WGSL features.

Stabilized features will be grouped in periodic releases
with backwards compatibilty with the aim of
nurturing an ecosystem of wgsl libraries.

## Filename extension

Source files containing
the community variant of wgsl-enhanced files
should have the `.wesl` extension
to distinguish them from vanilla W3C WGSL.

## WGSL Enhancements Roadmap

Several other features are queued up for discussion.
`import` is the first priority, the others are not prioritized.

Many projects already need these features:

* **[imports](./Imports.md)** -
`import` enhanced WGSL fragments from other files and eventually from libraries.

* **[conditional compilation](./ConditionalCompilation.md)** -
Select blocks of WGSL to enable based on external variables.
e.g. `#if MOBILE_CLASS_GPU`
or `#[cfg gpu_class="mobile")]`.

* **[simple templating](./SimpleTemplating.md)** -
Insert runtime
variables into WGSL from Rust/JavaScript host code into WGSL. e.g. `#{WORKGROUP_SIZE}`,
`#define`.

* **[packaging](./Packaging.md)** -
Package enhanced WGSL for crates.io and npm.

Several projects have identified these features as valuable to be shared more broadly:

* **[visibility control](./Visibility.md)** -
Define the public parts of wgsl.
e.g. `export`, `public`.

* **[generics](./Generics.md)** -
Define functions that work on any data type.

* **[virtual functions](./VirtualFunctions.md)** -
Customize a library by replacing a function.

Other features to consider include:

* **[extends](./Extends.md)** - Struct inheritance.

* **[versioning](./Versioning.md)** - Feature levels for enhanced WGSL, e.g. `version WESL2024`.

## Relationship to W3C WGSL and WebGPU

Enhanced WGSL features are invisible to WebGPU engines
like Dawn and wgpu.
All WGSL enhancements are translated to vanilla WGSL
before being passed to WebGPU calls
such as [`createShaderModule()`](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createShaderModule).

Hopefully community WGSL enhancements will
help inform future versions of W3C standard WGSL.
But designing changes to W3C standard WGSL is
beyond the scope of this project.
