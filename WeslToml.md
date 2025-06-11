# `wesl.toml` file format

WESL shader projects have a standardized `wesl.toml` file, which describes the project setup. It is used by language servers and linkers[^1].

The format looks as follows. As much of it is optional as possible.

```toml
[package] # Following the toml convention of grouping properties.

# Optional, used for specifying the name within wesl shaders.
# After all, the package manager's naming convention might clash with the WGSL naming conventions.
# (E.g. camelCase vs snek_case)
# Also deals with names like `@angular/animations`. Solves https://github.com/wgsl-tooling-wg/wesl-spec/issues/76
name = "angular_animations" 

# Not optional. Rust initially made it optional and it was a footgun.
# It ended up being too easy to opt into the semantics of the oldest Rust edition.
edition = "2025"

# Description and version fields do not exist.
# They match the package.json/Cargo.toml.
# Semantics are "we are publishing a normal package that happens to also contain wesl code".
# This lets us support wesl packages that come with host code!

# Where are the shaders located. This is the path of `package::`.
# We watch this directory for changes.
root = "./shaders"

# Some projects have large folders that we shouldn't react to. Globs for excluding them.
exclude = [ "**/test" ]

# Can be auto-inferred from the existence of a package.json.
# Necessary when both a `package.json` and a `Cargo.toml` are present.
# Inclusion of this field is gently encouraged.
package-manager = "npm"

# Optional dependencies section, automatically obtained from the package manager.
[dependencies]
 # Has no effect.
foolib = {}
 # Can be used for renaming packages. Now bevy in my code is called "cute_bevy".
cute_bevy = { package = "bevy" }
# Local imports, necessary for workspaces made of multiple separate packages.
mylib = { path = "../mylib" }
```


## FAQ

### Why restrict yourself to exactly one package manager in the `package-manager` field?

It is unreasonable to require a language server to check both package managers for the same dependencies, and run everything twice.

Tools are, however, free to support this sort of dual-publishing. They would have one primary package manager, and then attempt to mirror the structure for other package managers.

### How do language servers find dependencies?

Language server automatically obtain dependencies by asking the Typescript/Rust language servers.

Alternatively, a language server can implement the whole resolution algorithm. This seems feasible for Javascript, and significantly harder for Cargo.

In some cases, a language server can also obtain dependencies by calling the package manager's CLI.

Each wesl implementation will need to work together with language server authors.

### How does wesl find dependencies?

Wesl implementations get the dependencies this by running a build script.

In Rust, this means running a `build.rs` or a proc macro.

In Javascript, there are multiple flavours of build scripts. 
- Vite/rollup/... plugins
- A Node.js script in the user's project, which ends up having the same semantics for dependency resolution


## Defaults when `wesl.toml` is missing

It is possible to have WESL shaders without a `wesl.toml`. In that case, we default to

- All optional fields work as described above
- `root` is undefined, therefore the `package::` syntax cannot be used. Only relative imports, like `super::foo` are supported
- The edition field defaults to ... ?


[^1]: Linkers are free to offer their own configuration, in addition to supporting the `wesl.toml` format.
For example, wesl-js in the browser would be configured via a roughly equivalent JSON object. This lets wesl-js avoid bundling a whole toml file format parser.
