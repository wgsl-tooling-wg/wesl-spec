# Visibility Control

(TBD)

This section will describe wgsl enhancements to control which WGSL elements are visible to importers.

* which wgsl elements are available to import from wgsl modules within the same package?
* which wgsl elements are available to import from wgsl modules in other packages?
* how to re-export elements so that they're visible with a different path or name?
* lib.wgsl to make things visible at the root of a package?
* controlling host visible names like entry points and overrides?
* Should export allow `as` renaming?
* Treat .wgsl files as .wesl with every element exported?
* Why not export struct Foo?
  * Many current wgsl parsers (including wgpu's naga) would 
    choke on the unknown attribute as is and feels like having 
    two export forms is a bit inconsistent.

## Export

One natural extension is to add explicit exports.
For one, this would allow library authors to hide certain functions or structs from the outside world.
It would also enable re-exports, where a library re-exports a function from another library.

There are two variations of exports, which could be combined like in Typescript

### Exporting a list of items

A standalone export statement simply specifies which globals are exported.
Then, imports can be checked against the list of exports. This is very easy to parse and implement.

```
export { foo, bar };
```

And when one wants to export everything defined in the current file, they can use the `*` syntax.

```
export *;
```

To re-export an item, one can use the same syntax.

```
import my/lighting/{ pbr };

export { pbr };
```

### Exporting as an attribute

Exports can also be added as an attribute to the item itself.

```
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

Tools that look at source code will refer to a `package_root` in `wgsl.toml` that defines
the common prefix of `.wesl` and `.wgsl` files.

Source directories and files under the `package_root` and map directly to module paths
under the package name as expected.
e.g.

* Source file `C:\Users\lee\myProj\wgpu\foo.wesl` contains `export fn bar() {}`
* `wgpu` is the `project_root` in `wgsl.toml`.
* The project as published as package `fooz`.
* Other projects can write `import fooz/foo/bar;` to use `bar()`.

### lib.wgsl

If there is a file named `lib.wgsl` in the `package_root` directory,
any public exports in `lib.wgsl` are visible at the root of the module.
e.g. if a package ‘pkg’ has a source file `pkg_root/lib.wgsl`
that contains `export fn fun()`,
a module file in another package can import that function with `import pkg/fun`.