# WESL - A Community Standard for Enhanced WGSL

We propose building an enhanced version of WebGPU's
[WGSL](https://www.w3.org/TR/WGSL/) shading language
using community tooling.
We want to see WGSL enhancements shared more broadly.
We want to enable a WGSL library ecosystem.

Many community projects have independently uncovered
a need for WGSL enhancements like module composition,
conditional compilation, and simple templating.
Fancier features like generics are also of interest in the community.
By evolving a shared community set of WGSL enhancement definitions,
we can build common tools.

Better tooling benefits everyone,
including smaller projects that can't afford to build
their own enhancements and associated tools.
And a shared standard can enable resusable WGSL libraries
on npm and crates.io.

Join us on [github](https://github.com/wgsl-tooling-wg/wesl-spec)
or [discord](https://discord.gg/Ng5FWmHuSv) to help.

## Goals

Our ambition is to create a modestly improved, practical variant of WGSL
called WESL (WGSL Enhanced Shading Language, pronounced like 'weasel').

WESL should feel like WGSL with a few useful things added.

## WGSL Enhancements

We're aiming to prioritize WGSL enhancements that are: 1) important for community projects,
2) feel natural to the WGSL programmer,
and 3) not too difficult to integrate into community tools.

The simple enhancements we want to find will take some time to stabilize.
Adventerous projects will be able to opt in to experimental enhancements
to try alongside stable features.

A WESL enhancement for `import` statements
to enable importing WGSL elements like functions from other files is our first priority to
stabilize. See spec draft here: **[imports](./Imports.md)**.

Several other WGSL enhancements
are on the [roadmap](#wgsl-enhancements-roadmap).

## Tools

We'll build and promote tools to support community standard WGSL.

**Linking** - Tools to bundle WESL files together
while translating WESL into vanilla W3C WGSL.

**Test Cases** - A collection of examples to help tools interoperate.

**WESL Language Servers** - Enable syntax directed editing in IDEs.
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
nurturing an ecosystem of enhanced wgsl libraries.

## Filename extension

Source files containing
WESL files
should have the `.wesl` extension
to distinguish them from vanilla W3C WGSL.

## WGSL Enhancements Roadmap

Several other features are queued up for discussion.
`import` is the first priority, the others are not prioritized.

Many projects already need these features:

* **[imports](./Imports.md)** -
`import` WESL code from other files and eventually from libraries.

* **[conditional compilation](./ConditionalCompilation.md)** -
Select blocks of WESL to enable based on external variables.
e.g. `@if(mobile_gpu)`

* **[simple templating](./SimpleTemplating.md)** -
Insert runtime
variables into WGSL from Rust/JavaScript host code into WGSL. e.g. `#{WORKGROUP_SIZE}`,
`#define`.

* **[packaging](./Packaging.md)** -
Package enhanced WGSL for crates.io and npm.

Several projects have identified these features as valuable to be shared more broadly:

* **[visibility control](./Visibility.md)** -
Define the public parts of WESL.
e.g. `export`, `public`.

* **[generics](./Generics.md)** -
Define functions that work on any data type.

* **[virtual functions](./VirtualFunctions.md)** -
Customize a library by replacing a function.

Other features to consider include:

* **[extends](./Extends.md)** - Struct inheritance.

* **[versioning](./Versioning.md)** - Feature levels for WESL, e.g. `enable wesl_2025`.

For guidance on contributing
to the design of WGSL extensions, see **[Designing](./Designing.md)**.

## Relationship to W3C WGSL and WebGPU

WESL enhancements features are invisible to WebGPU engines
like Dawn and wgpu.
All WESL enhancements are translated to vanilla WGSL
before being passed to WebGPU calls
such as [`createShaderModule()`](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createShaderModule).

Hopefully community WESL enhancements will
help inform future versions of W3C standard WGSL.
But designing changes to W3C standard WGSL is
beyond the scope of this project.
