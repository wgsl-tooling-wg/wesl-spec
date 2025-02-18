# Name mangling

Linker will mangle names internally, but we're hoping to keep the mangling details out of the public api.

See also renaming host visible names in [Visibility](./Visiblity.md).

(TBD)

* Extend scheme to support generics.
* Can we find an explicit naming scheme where packages specify names for their host exposed WESL parts
  rather than publishing automatically mangled names?
  * e.g. `@publicName("mypkg_util_bar") @compute fn compute_shader() {}`
  * that way the host code doesn't need to change if the WESL file structure is refactored.
 
## What gets mangled

All declaration identifiers get mangled, except:
* root module declarations (this is being discussed, see <https://github.com/wgsl-tooling-wg/wesl-js/issues/98>)
* declarations imported in the root module
  * if the import renames the declaration (using the `import foo as bar` syntax), the declaration is renamed.
  * if the import is a module (`import some::module` where `some/module.wesl` exists) it does not affect mangling.

## Mangling

To join multiple modules into one, we need to make sure that names do not collide. This is done by name mangling.

Mangling is the process of renaming a declaration and its references to avoid name collisions.
It is not easy: because of name shadowing, local-scope declarations can have the same name as module-scope ones.
To rename all references to a declaration, one has to walk the syntax tree while keeping track of the scope (list of local declarations).

Compiled WGSL files have a few exposed parts, namely the names of the entry functions and pipeline overridable constants (together called "host-visible names").
These need to be stable and predictable, so we will introduce a stable, human-readable and reversible name mangling scheme.

For name mangling, we introduce the concept of a **absolute module identifier**.
This is an ID that is relative to the nearest _package_, and is unique for each module.
For example, given a file `my/geom/sphere.wgsl`, and a package `my`, then the absolute module identifier is `my_geom_sphere`.
It is always this, regardless of how the file is imported.
If a file or folder name already contains an underscore, it is replaced by two underscores.

The name mangling scheme is as follows:

1. Underscores are used as separators. If a name already contains underscores, they are each replaced by two underscores.
2. Prefix each name with the absolute module identifier, followed by a single underscore.

For example, given a file `my/geom/sphere.wgsl` with a function `draw_now`, the mangled name would be `my_geom_sphere_draw__now`.


Example implementation (non-normative)
```js
/** @param qualified_name string[] */
function mangle(qualified_name) {
    return qualified_name.map(write_escaped).join('_');
}
/** @param qualified_name string[] */
function write_escaped(text) {
    return text.split('').map(c => c == '_' ? '__' : c).join('');
}

mangle(["bevy_pbr", "lighting", "fragment_main"])
// returns "bevy__pbr_lighting_fragment__main" 
```

### Unmangling

We keep track of a stack of parts, and always add characters to the last part. It starts with a single empty part.

To unmangle a name, one goes over a mangled name from left to right.  
* If two underscores are encountered, we add one underscore to the current part, and skip the second underscore.  
* If only one underscore is encountered, we start a new part.
* If anything else is encountered, we add it to the current part.

At the end, the final part is the item name, and the other parts are the module path.

