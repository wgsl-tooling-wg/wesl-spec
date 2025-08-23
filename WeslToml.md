# `wesl.toml` file format

WESL shader projects have a standardized `wesl.toml` file, which describes the project setup. It is used by tools such as language servers and linkers[^1].

The format looks as follows. As much of it is optional as possible.

```toml
[package]
# Version of WESL used in this project.
edition = "unstable_2025"

# Optional, can be auto-inferred from the existence of a package manager file.
# Inclusion of this field is encouraged.
package-manager = "npm"

# Where are the shaders located. This is the path of `package::`.
# We watch this directory for changes.
root = "./shaders"

# Optional
include = [ "shaders/**/*.wesl", "shaders/**/*.wgsl" ]

# Optional.
# Some projects have large folders that we shouldn't react to. 
exclude = [ "**/test" ]

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

For user applications, feel free to start without a `wesl.toml`. We'll assume reasonable defaults, which will evolve over time.

As your application grows, we suggest adding a `wesl.toml` for better consistency. This lets you pin down the exact behavior of wesl.

### `edition` field

Specifies which edition is used by wesl. All wesl editions can be used.

- Mandatory
  - Falls back to latest edition that tools know.
- Currently only `unstable_2025` is accepted, since we do not have an edition yet.
  - No incompatible changes are planned, but we have not committed to stable version for long term support.
- Dependencies are interpreted using the edition that they declared, not the edition of the consuming package.

### `package-manager` field

Which package manager is used for resolving wesl libraries.

- Optional, but encouraged.
  - Can be inferred from the existence of certain files (`package.json` and `Cargo.toml`)
  - Necessary when multiple such files are present.
- `npm` and `cargo` are accepted.

It is limited to one package manager to reduce implementation complexity.
We do not have any wesl implementations that would allow for mixing and matching packages from different package managers.
For dual publishing, the expectation is that one would have a primary package manager, and then attempt to mirror the structure for other package managers.

### `root` field

Specifies the root folder or root file for the `package::` syntax.

- Optional
  - Defaults to `path of package manager file/shaders`
 
 For ecosystems where there is no such package manager file, this is mandatory. 
 Currently we do not have any such cases.

## `include` and `exclude` fields

Specifies an array of patterns where wesl files are located. They patterns are relative to the `wesl.toml` file.

The default value of `include` is all wesl and wgsl files in the `root` directory, recursively.
The default value of `exclude` is an empty array.

The patterns are glob patterns, which support
- `*` for zero or more characters in file/folder names
- `?` for one character in a file/folder name
- `**/` for any directory nested to any level

The globs must match file names, and the last part of the patterns must contain a file extension.

### `dependencies` entry

All used dependencies are explicitly listed here.

Its structure is `wesl_name = { package = "name_used_in_package_manager" }`. It supports `package` or `path` specifications:

 - `package` accepts the name of a library installed with npm or Cargo.
 - `path` accepts a relative file path to a directory containing a `wesl.toml` file. The path is relative to the current `wesl.toml`.

This lets us additionally deal with names that would be impossible to spell in WESL.
e.g. `fun_shaders = { package = "@lorem/fun_shaders" }`

### `dependencies = "auto"`

Instead of specifying a list of dependencies, one can specify `dependencies = "auto"`.
Then, the available dependencies are automatically inferred from the list of libraries installed with npm or Cargo.

This is an ecosystem specific feature. It is supported by:

- npm: enabled by default. Exact semantics will be figured out.


## Q&A

### Description and version fields

Description and version fields do not exist.
Instead, we use thepackage.json/Cargo.toml.
Semantics are "we are publishing a normal package that happens to also contain wesl code".
This lets us support wesl packages that come with host code!

