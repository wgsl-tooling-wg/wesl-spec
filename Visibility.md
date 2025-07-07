# Visibility Control

This section will describe WESL enhancements to control which WGSL elements are visible to importers.

* how to re-export elements so that they're visible with a different path or name?
* controlling host visible names like entry points and overrides?
* Should export allow `as` renaming?
* Why not export struct Foo?
  * Many current WGSL parsers (including wgpu's naga) would
    choke on the unknown attribute as is and feels like having
    two export forms is a bit inconsistent.
* Consider changing to public by default for imports within the package only.
  * Matches semantics when importing from WGSL code
  * no annotation effort for tiny projects, everything public is fine.
  * (Private by default gives a gentle push to programmers to consider their api every time
    they add a public annotation.)
  * (And the path of laziness leads to undersharing,
    which is safer from a maintenance point of view.)
  * (less consistent with package visibility.)
  * (unexpected if programmers are accustomed to e.g. JavaScript imports.)

## Export

One natural extension is to add explicit exports.
For one, this would allow library authors to hide certain functions or structs from the outside world.
It would also enable re-exports, where a library re-exports a function from another library.

There are two variations of exports, which could be combined like in Typescript

### Exporting a list of items

A standalone export statement simply specifies which globals are exported.
Then, imports can be checked against the list of exports. This is very easy to parse and implement.

```wgsl
export { foo, bar };
```

And when one wants to export everything defined in the current file, they can use the `*` syntax.

```wgsl
export *;
```

To re-export an item, one can use the same syntax.

```wgsl
import my/lighting/{ pbr };

export { pbr };
```

### Exporting as an attribute

Exports can also be added as an attribute to the item itself.

```wgsl
@export
struct Foo {
    x: f32;
}

@export
fn foo() {}
```

This is more user friendly, but also more complex to parse. It requires a partial parsing of the WGSL file to find the exports and their names.
A future export specification would include the minimal WGSL syntax that is necessary to implement this.

## Translating Source File Paths to Import Module Paths

Tools that look at source code will refer to a `package_root` in `wesl.toml` that defines
the common prefix of `.wesl` and `.wgsl` files.

Source directories and files under the `package_root` and map directly to module paths
under the package name as expected.
e.g.

* Source file `C:\Users\lee\myProj\wgpu\foo.wesl` contains `export fn bar() {}`
* `wgpu` is the `project_root` in `wesl.toml`.
* The project as published as package `fooz`.
* Other projects can write `import fooz/foo/bar;` to use `bar()`.

### `lib.wgsl`

If there is a file named `lib.wgsl` in the `package_root` directory,
any public exports in `lib.wgsl` are visible at the root of the module.
e.g. if a package ‘pkg’ has a source file `pkg_root/lib.wgsl`
that contains `export fn fun()`,
a module file in another package can import that function with `import pkg/fun`.

## Libraries and Internal Modules Need Privacy

We want library publishers to decide carefully what to expose as
part of their public api, so they can upgrade private parts of the library safely.
Smooth upgrades are valuable to the library ecosystem.

Authors of significant internal modules will similarly want
to make a public vs private distinction to reduce maintenance effort as
the internal module evolves.

### Private by Default

We propose that importable elements like functions and structs
be private by default, (i.e. inaccessible from import statements).
The programmer needs to add an annotation to make them public, (i.e. available to import).
The programmer can decide whether the element should be public within the package only
or also public to importers from other packages. Perhaps `@export` and `@export(public)`.

Importable elements from (unenhanced) WGSL code may be imported from WESL functions
in the same package. Elements in WGSL code are not public from other packages
(WESL code may reexport WGSL element for package publishing).
