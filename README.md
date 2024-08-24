# A Community Standard for Enhanced WGSL

We propose specifying enhancements
to WebGPU's [WGSL](https://www.w3.org/TR/WGSL/) shading language to support
features like [`import`](./Imports.md), [`export`](./Export.md)
and [conditional compilation](./ConditionalCompilation.md).
We propose building and promoting community tools that support
a standard community enhanced version of WGSL.

By standardizing an enhanced WGSL, we can
share the effort of building better tools for everyone,
including small projects
that can't afford to create their own tooling.
And with a standardized version of community WGSL,
the community can publish WGSL libraries in crates.io and npm.

Join us on [github](https://github.com/wgsl-tooling-wg/wgsl-import-spec)
or [discord](https://discord.gg/FXhZDV8V) to help.

## WGSL Enhancements

We're aiming to prioritize WGSL enhancements that are: 1) important for community projects,
2) feel natural to the WGSL programmer,
and 3) reasonable to integrate into tools.

A WGSL enhancement to support `import` statements is planned first
to enable importing WGSL elements like functions from other files.
See spec draft here: **[imports](./Imports.md)**.

Several other WGSL enhancements
are in various stages of discussion,
see [below](#other-wgsl-enhancements).
We'll group a first set of stable enhancements into a 1.0 release.

## Tools

We'll build or promote tools to support a community standard WGSL.

**Linking** - Tools to bundle wgsl files together
while translating extended WGSL into vanilla W3C WGSL.

**Test Cases** - A collection of examples to help tools interoperate.

**WGSL Language Servers** - Enable syntax directed editing in IDEs.
Shader development with IDEs should be well supported.

**Syntax Highlighters** - Display pretty extended WGSL on web pages with
tools like Shiki and CodeMirror.

**More Tools** - We'd like to see documentation extractors (rustdoc/javadoc),
prettifiers, etc. that understand the community standard version of WGSL.

## Stability

As a community project,
we can iterate quickly on experimental WGSL features to use
in local projects, enabled by an `experimental` flag.

As mention Stabilized features will be grouped in periodic releases
with backwards compatibilty
so that an ecosystem of enhanced wgsl libraries can prosper.

## Filename extension

Many tools should be able to handle extended wgsl transparently.
But for clarity, we expect a separate file extension for enhanced wgsl files,
perhaps `.wesl`.

## Other WGSL Enhancements

Several other features are in various stages of discussion.

**[conditional compilation](./ConditionalCompilation.md)** -
Select blocks of WGSL to enable based on external variables. e.g. `#if MOBILE_GPU`.

**[variable substitution](./VariableSubstiution.md)** -
Insert runtime
variables from Rust/js host code into WGSL. e.g. `#{WORKGROUP_SIZE}`.

**[generics](./Generics.md)** -
Define functions that work on any data type.

**[packaging](./Packaging.md)** -
Package WGSL for crates.io and npm.

**[virtual functions](./VirtualFunctions.md)** -
Customize a library by replacing a function.

**[visibility control](./Visibility.md)** -
Define the public parts of wgsl.
e.g. `export`.

**[extends](./Extends.md)** - Struct inheritance.

**[versioning](./Versioning.md)** - specify feature levels for enhanced WGSL, e.g. `version WESL2024`.

## Relationship to W3C WGSL and WebGPU

Enhanced WGSL features are invisible to WebGPU engines -
all WGSL enhancements are translated to vanilla WGSL before being passed to WebGPU
call like [`createShaderModule()`](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createShaderModule).

Hopefully some community WGSL enhancements will be relevant to
guide future versions of W3C standard WGSL.
That would be welcome, but it is explicitly _not_ a goal.
