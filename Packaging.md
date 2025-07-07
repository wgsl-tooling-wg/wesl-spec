# Packaging

(TBD)

The section will discuss packaging WESL in reusable crates or npm packages.

See also [Visibility](Visibility.md).

* What goes in `package.json`?
  * at runtime in the browser, the linker needs
    WESL strings and relative paths for every needed WESL file, from every every package.
    * (this so the linker can resolve imports)
  * something needs to tell javascript bundlers to include needed WESL files in the bundle:
    * we could ask package publishers to produce a javascript file containing a map of relative paths and WESL strings
    * perhaps better would be to have a vite/rollup plugin trace from the source WESL to the needed package WESL files (using the linker as a library to parse WESL)
      * this would avoid requiring publishers to manually publish string maps of WESL text,
        instead publishers would just publish files in dist.
        * the consumer plugin tool would bundle files into strings.
      * corner case issue: runtime conditional compilation could potentially change the imported
        files required. Perhaps we'll need some workaround markers in this rare case, so tree shaking can still work.
    * note that the WESL-linker already has this problem of bundling local WGSL files
      and asks users to 'self package' WGSL using import.meta.glob
      (which bundles files into [file path, WGSL string] pairs)
  * what other metadata should be available to the linker?
    * name of package.
    * host visible bits from WGSL? e.g. entry points, runtime variables, overrides, binding groups, uniforms?
      An IDE tool could use that data to typecheck/autocomplete host calls.
      Things like entry points are computable from the WGSL,
      but it's a lot to ask of a language server to parse through all the WGSL in advance to find them.
      Probably better to put it into the metadata if we need it.
      * these are parts of the API of the package as a whole and can reduce bugs by making them visible
  * how do we handle packages with multiple entry points e.g. stoneberry/reduce stoneberry/prefixSum.
* What goes in `cargo.toml`?

## `wesl.toml` file

The `wesl.toml` file is a configuration file that tells the language server where to find the packages. It is similar to a `Cargo.toml` file in that regard.

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

### Unresolved questions

* Is .toml the best file format for the `wesl.toml`? Some alternatives would be JSON/JSON5 and StrictYAML.
* do we need to put paths for every package's WGSL in `wesl.toml`?
  * Fine to get started that way if need be, but its a maintenance burden every user..
  * Better if we could get the language servers to find the WGSL in packages in node_modules.
* Do we need to list the WGSL package names in `wesl.toml`?
  They'll already be listed in as `cargo.toml` / `package.json`..
