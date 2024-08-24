# Packaging

The section will discuss packaging extended wgsl in reusable crates or npm packages.

See also [Visiblity Control](Visiblity.md).

(TBD)

* What goes in `package.json`?
* What goes in `cargo.toml`?

## `wgsl.toml` file

The `wgsl.toml` file is a configuration file that tells the language server where to find the packages. It is similar to a `Cargo.toml` file in that regard.

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

- Is .toml the best file format for the `wgsl.toml`? Some alternatives would be JSON/JSON5 and StrictYAML.