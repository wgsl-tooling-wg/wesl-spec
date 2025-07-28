# `wesl.toml` file format

WESL shader projects have a standardized `wesl.toml` file, which describes the project setup. It is used by tools such as language servers and linkers[^1].

The format looks as follows. As much of it is optional as possible.

```toml
[package]
# Version of WESL used in the local shaders
edition = "unstable_2025"

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

# Lists all used dependencies
[dependencies]
# Shorthand for `foolib = { package = "foolib" }`
foolib = {}
 # Can be used for renaming packages. Now bevy in my code is called "cute_bevy".
cute_bevy = { package = "bevy" }
# File path to a folder with a wesl.toml. Simplest kind of dependency.
mylib = { path = "../mylib" }
```


[^1]: Linkers are free to offer their own configuration, in addition to supporting the `wesl.toml` format.
For example, wesl-js in the browser would be configured via a roughly equivalent JSON object. This lets wesl-js avoid bundling a whole toml file format parser.


## Semantics

A `wesl.toml` file is mandatory for libraries.
For user applications, we allow it to be missing, and try to choose reasonable defaults.
However, this can lead to different tools assuming different values. 

### `edition` field

Specifies which edition is used by wesl. All wesl editions can be used.

- Mandatory
  - Falls back to latest edition that tools know.
- Currently only `unstable_2025` is accepted, since we do not have an edition yet.
  - No incompatible changes are planned, but we have not committed to stable version for long term support.

### `package-manager` field

Which package manager is used for resolving wesl libraries.

- Optional, but encouraged.
  - Can be inferred from the existence of certain files (`package.json` and `Cargo.toml`)
- `npm` and `cargo are accepted.

It is limited to one package manager to reduce implementation complexity.
We do not have any wesl implementations that would allow for mixing and matching packages from different package managers.
For dual publishing, the expectation is that one would have a primary package manager, and then attempt to mirror the structure for other package managers.

## `include` and `exclude` fields

The `include` and `exclude` fields support globs. The paths are relative to the `wesl.toml` file.

The default value of `exclude` is an empty array.

### `dependencies` entry

All used dependencies are explicitly listed here.

Its structure is `wesl_name = { package = "name_used_in_package_manager" }`

This lets us additionally deal with names that would be impossible to spell in WESL.
e.g. `fun_shaders = { package = "@lorem/fun_shaders" }`

### `dependencies = "auto"`

Instead of specifying a list of dependencies, one can specify `dependencies = "auto"`.
Then, the available dependencies are automatically inferred from the list of installed libraries. 

This is an ecosystem specific feature. It is supported by

- npm: enabled by default. Exact semantics will be figured out.


## Q&A

## Description and version fields

Description and version fields do not exist.
Instead, we use thepackage.json/Cargo.toml.
Semantics are "we are publishing a normal package that happens to also contain wesl code".
This lets us support wesl packages that come with host code!

### How does wesl resolve dependencies?

Wesl implementations resolve the dependencies by running a build script.

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
