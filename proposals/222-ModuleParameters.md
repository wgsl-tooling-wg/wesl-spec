# Module Parameters

- **Status**: Proposal
- **Discussion**: [#222](https://github.com/webgpu-tools/wesl-spec/issues/222)
- **Extension marker**: `wesl_module_parameters`
- **Implementations**: none
- **Replaces / extends**: [#146](https://github.com/webgpu-tools/wesl-spec/issues/146), [#164](https://github.com/webgpu-tools/wesl-spec/issues/164); supersedes the unspec'd `constants::` virtual module

See also [ModuleParametersDesign.md](222-ModuleParametersDesign.md) (design background).

## Declaring module parameters

```wgsl
// lights.wesl
module <const MAX: u32 = 8>;

struct Light { pos: vec3f, color: vec3f }

var<private> list: array<Light, MAX>;
var<private> count: u32;

fn push(l: Light) { list[count] = l; count += 1u; }
```

- parameters go in a `module` statement near the top of the file. 
- module params are named, typed, and can have defaults.
- inside the module, the parameter works as an ordinary `const`. (and is emitted as a WGSL `const`)
- params with no defaults must be set from host code or via import.

Details
- multiple parameters can be separated by commas. 
- `module` statements may not be inside an `@if` condition.
- `public` marks a parameter as settable from other packages; unmarked parameters are package-internal.

## Conditions

```wgsl
module <public const SAMPLES: u32 = 8>;

@if(SAMPLES > 4)
fn blurWide(/* ... */) { /* ... */ }
```

- `@if`/`@elif` take a translate-time constant expression of type `bool`; identifiers in the expression are module parameters or consts. (condition variables no longer have a separate namespace)
- A const in an `@if` body can be used in a different condition.
- A condition or const that transitively depends on itself is an error.

## Importing

```wgsl
import package::lights<MAX = 32>::{Light, push};

import random::rng<SEED = 1> as r1;
import random::rng<SEED = 2> as r2;      // two independent instantiations of rng
import random::rng<SEED = 1+1> as r3;    // alias for r2 (not a third rng)
```

- arguments are named, not positional. unmentioned parameters take their declared defaults.
- a plain import (no `< >`) instead takes the link-wide values
- priority for plain imports: host set default, main module reset default, declaring module default.
- libraries generally use plain imports so that apps can configure
- `< >` reads as specialization, like WGSL type parameters
- const values are const-eval'd

## Module instantiation / concretization when emitting WGSL

When we emit WGSL, we emit a duplicated set of declarations for each set of parameters in use,
with the exception that types are shared where possible. 

- all `var` declarations are forked per param set
- types are forked if they depend on a parameter transitively, otherwise shared

Details:
- implementations can optionally reduce emitted code size by identifying declarations
  that don't need to be forked (e.g. functions that don't transitively reference
  a parameter or a forked `var` needn't be duplicated)
- [ModuleParametersDesign](./222-ModuleParametersDesign.md) has more examples.

## Module parameters in the main module

- `module` parameters in a main app module are the (only) module params the host can see.
  (libraries remain encapsulated)
- The main app module can expose library module parameters to the host
- The main app module can reset link-wide defaults
- `private` on an entry sets the value for the link but hides it from the host
- `@main module` is required for main modules that reset param defaults or expose params from another module to the host. 
  (And recommended for other main modules. But e.g., a previewer can link and play any module as the main module, so it's okay to link from a main module not tagged `@main`).

```wgsl
// main.wesl
@main module <
  const DEBUG = false,                // param declared locally
  lygia::tonemap::GAMMA,              // expose a library param to host code
  bevy_pbr::config::MAX_LIGHTS = 32,  // reset a default
  private random::rng::SEED = 47,     // reset default, hidden from host
>;
```

Host code can set values for main module parameters, e.g.:

```ts
await link({ ..., params: { DEBUG: true, MAX_LIGHTS: 16 } });
```

## Laziness and cycles

- imports remain lazy, unused imports are not instantiated.
- import cycles are still allowed as now, but a referenced import cycle whose arguments don't stabilize around the loop is an error. (a module importing a fixed instantiation of itself is fine)
- Conditions and consts evaluate on demand, not in separate passes (so that conditions can depend on consts in other conditions)

```wgsl
// bar.wesl
module <const C: u32 = 0>;

import package::bar<C = C + 1>::f;   // error if f is used
```

## Future

- type parameters: `module <T: VecF = vec2f>`.
- function parameters: `module <warp: fn(p: T) -> T = identity>`
- named impls: `module <n: #Noise = Simplex>`
- reflection: report the main module's parameters to host code (with types and defaults)
