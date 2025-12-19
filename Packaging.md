# Packaging

WESL enables shader packages for reusing shader code by other packages or applications. Shader packages are published to repositories such as [npm] or [crates.io].

## Using shader packages

Dependencies for your application are defined in the [`wesl.toml`] file. The WESL linker is responsible for finding and downloading dependencies: [wesl-js] will fetch packages from [npm] and [wesl-rs] will fetch them from [crates.io].
Inside your shader modules, you can reach declarations in dependencies with import paths, as explained in [imports].

## Creating and publishing shader packages

Any [`wesl.toml`] file declares a new shader package, which can be published by the linker's CLI.
Publishing packages is implementation-specific, follow the instructions for your linker ([wesl-js] or [wesl-rs]).

### Package naming guidelines

You can publish your package with whatever name you like. We however recommend following these recommendations so your package can be easily found by searching the package registry.

* Use a name that is also a valid WGSL identifier. Otherwise, end-users will have to rename it in [`wesl.toml`].
* Add the `_wgsl` suffix to the name: `mypackage_wgsl`.
* Use snake_case (underscores to separate words): `my_great_package_wgsl`.
* If your package is part of a larger project, or produced by a company, you can prefix it with that name: `mycompany_mypackage_wgsl`.
  * For [wesl-js] specifically, you can use the common naming convention `@mycompany/mypackage_wgsl`
* Look for packages with the same name in both [npm] and [crates.io].
  * It is courtesy to leave the name free for the original author if they wish to publish to the other registry.
  * It also avoids confusion for end-users who may think it is the same package.

## Semver-compatibility and dependency unification

(TODO: unification with param const?)

If two packages in the dependency tree are [semver-compatible](https://semver.org/), the WESL linker will unify them, meaning it will include only one version of the package in its output (usually the highest semver-compatible version available).
This unification can have observable side-effects that a user must be warry of. Module-scope declarations may, or may not be duplicated.

### Example

```wgsl
// package random
// --------------
var<private> prng_state: f32 = 0;

// return a random float in [0, 1]. Do not actually use this function, it is awful.
fn rand() -> f32 {
    prng_state = fract(sin(x)*12345.6789);
    return prng_state;
}
```

Here, the `random::rand()` function has internal state represented by `prng_state`.
If two packages depend on semver-compatible versions of `random`, then they would share that internal state.
If however they depend on non-semver-compatible versions, two copies of the state and the `rand()` function will be used, suddenly producing different results.

## Visibility

All declarations reachable from a direct dependency's root module can be accessed from the parent package.
Declarations in indirect dependencies (i.e., dependencies of dependencies) are never reachable.

Future iterations of WESL may introduce a re-export and/or visibility control mechanism. See [visibility].


[npm]: https://www.npmjs.com/
[crates.io]: https://crates.io/
[`wesl.toml`]: WeslToml.md
[imports]: Imports.md
[visibility]: Visibility.md
[wesl-js]: https://github.com/wgsl-tooling-wg/wesl-js
[wesl-rs]: https://github.com/wgsl-tooling-wg/wesl-rs
