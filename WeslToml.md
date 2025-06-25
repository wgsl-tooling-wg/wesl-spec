# `wesl.toml` file format

WESL shader projects have a standardized `wesl.toml` file, which describes the project setup. It is used by language servers and linkers[^1].

The format looks as follows. As much of it is optional as possible.

```toml
[package]
# Which wesl edition are we using
edition = "unstable_2025"

# Description and version fields do not exist.
# They match the package.json/Cargo.toml.
# Semantics are "we are publishing a normal package that happens to also contain wesl code".
# This lets us support wesl packages that come with host code!

# Where are the shaders located. This is the path of `package::`.
# We watch this directory for changes.
root = "./shaders"

# Optional. Default value is all wesl and wgsl files in the root directory.
include = [ "shaders/**/*.wesl", "shaders/**/*.wgsl" ]

# Optional.
# Some projects have large folders that we shouldn't react to. 
exclude = [ "**/test" ]

# Optional, can be auto-inferred from the existence of a package.json.
# Necessary when both a `package.json` and a `Cargo.toml` are present.
# Inclusion of this field is encouraged.
package-manager = "npm"

# Automatically read dependencies from the package.json
# Only supported for some package managers!
auto-dependencies = true

# Lists all used dependencies
[dependencies]
# Shorthand for `foolib = { package = "foolib" }`
foolib = {}
 # Can be used for renaming packages. Now bevy in my code is called "cute_bevy".
cute_bevy = { package = "bevy" }
# Local imports, necessary for workspaces made of multiple separate packages.
mylib = { path = "../mylib" }
```


[^1]: Linkers are free to offer their own configuration, in addition to supporting the `wesl.toml` format.
For example, wesl-js in the browser would be configured via a roughly equivalent JSON object. This lets wesl-js avoid bundling a whole toml file format parser.


## Semantics

### `edition` field

Supports all wesl editions. Currently only `unstable_2025` is accepted, since we do not have an edition yet.

### `package-manager` field

Supports

- `npm`
- `cargo`

And it must be explicitly limited to one package manager to massively reduce implementation complexity.
We also do not have any wesl implementations that would allow for mixing and matching packages from different package managers.

For dual publishing, the expectation is that one would have a primary package manager, and then attempt to mirror the structure for other package managers.

## `auto-dependencies` field

The `auto-dependencies` field is supported for

- npm: enabled by default. 

It is not supported for other package managers and defaults to `false` for those.


## Q&A

### How do language servers find dependencies?

Language server automatically obtain dependencies by asking the Typescript/Rust language servers.

Alternatively, a language server can implement the whole resolution algorithm. This seems feasible for Javascript, and significantly harder for Cargo.

In some cases, a language server can also obtain dependencies by calling the package manager's CLI.

Each wesl implementation will need to work together with language server authors.

### How does wesl resolve dependencies?

Wesl implementations resolve the dependencies this by running a build script.

In Rust, this means running a `build.rs` or a proc macro.

In Javascript, there are multiple flavours of build scripts. 
- Vite/rollup/... plugins
- A Node.js script in the user's project, which ends up having the same semantics for dependency resolution


## Defaults when `wesl.toml` is missing

**Creating a `wesl.toml` is recommended.**

However, we support having WESL shaders without a `wesl.toml`. In that case, we default to

- All optional fields work as described above
- `root` is undefined, therefore the `package::` syntax cannot be used. Only relative imports, like `super::foo` are supported
- `edition` defaults to the latest wesl edition that *the tool* knows of.

Not specifying an edition comes with the explicit risk of *tools no longer understanding your wesl code after an update*.
