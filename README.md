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
For tool makers,
more users makes it easier to justify the extra effort
of creating more polished designs.

A standard enhanced WGSL also helps us to
share shader code.
We'd like to enable WGSL libraries
in crates.io and npm.

## WGSL Enhancements

We're aiming for to design and prioritize WGSL enhancements that are
important for community projects,
feel natural to the WGSL programmer,
and are reasonable to integrate into tools.

A WGSL enhancement to support `import` statements is planned first
to enable importing WGSL elements like functions from other files.
See spec draft here: **[imports](./Imports.md)**.

Several [other](#other-wgsl-enhancements) WGSL enhancements
are in various stages of discussion, see below.

## Tools

We'll build or promote tools to support a community standard WGSL.

**Linking** - Bundling tools to combine wgsl files together,
and translate extended WGSL into vanilla W3C WGSL.

**Test Cases** - A collection of examples to help tools be interoperable.

**WGSL Language Servers** Enable syntax directed editing in IDEs.
Shader development with IDEs should be well supported.

**Syntax Highlighters** Display pretty extended WGSL on web pages and in other tools.

**more** We'd like to see documentation extractors, prettifiers, etc. understand
a standard extended version of WGSL.

Work has begun on a few of these tools, we could use your help.
File a an issue, or visit on [discord](https://discord.gg/FXhZDV8V).

## Relationship to W3C WGSL and WebGPU

Enhanced WGSL features are invisible to WebGPU engines -
all WGSL enhancements are translated to vanilla WGSL before being passed to WebGPU
call like [`createShaderModule()`](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createShaderModule).

Hopefully some community WGSL enhancements will be relevant to
guide future versions of W3C standard WGSL.
That would be welcome, but it is explicitly _not_ a goal.

## Stability

The intent is to eventually maintain backward compatibility so that an ecosystem
of enhanced wgsl libraries can prosper.
Stabilized features will be grouped and released periodically,
analagously to ECMAScript and its `ES2024` feature level.

As a community project,
we can iterate quickly on experimental WGSL features to share
for use in local projects,
enabled by an `experimental` flag.

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

**[visibility control](./Visibility.md)** - Define the public parts of wgsl.
e.g. `export`.

**[extends](./Extends.md)** - Struct inheritance.

**[versioning](./Versioning.md)** - specify feature levels, ES2024
