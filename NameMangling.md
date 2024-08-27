# Name mangling

Linker will mangle names internally,
but we're hoping to keep the mangling details
out of the public api.

See also renaming host visible names in [Visibility](./Visiblity.md).

(TBD)

## Mangling
To join multiple modules into one, we need to make sure that names do not collide. This is done by name mangling.

Compiled WGSL files have a few exposed parts, namely the names of the entry functions and pipeline overridable constants. These need to be stable and predictable, so we will introduce a stable, predictable and human-readable name mangling scheme.

Finally, only importable items need to be mangled.

For name mangling, we introduce the concept of a **absolute module identifier**. This is an ID that is relative to the nearest _package_, and is unique for each module. For example, given a file `my/geom/sphere.wgsl`, and a package `my`, then the absolute module identifier is `my_geom_sphere`. It is always this, regardless of how the file is imported. If a file or folder name already contains an underscore, it is replaced by two underscores.

The name mangling scheme is as follows:

1. Underscores are used as separators. If a name already contains an underscore, it is replaced by two underscores.
2. Prefix each name with the absolute module identifier, followed by a single underscore.

For example, given a file `my/geom/sphere.wgsl` with a function `draw_now`, the mangled name would be `my_geom_sphere_draw__now`.

### Unmangling

We keep track of a stack of parts, and always add characters to the last part. It starts with a single empty part.

To unmangle a name, one goes over a mangled name from left to right.  
If two underscores are encountered, we add one underscore to the current part, and skip the second underscore.  
If only one underscore is encountered, we start a new part.
If anything else is encountered, we add it to the current part.

At the end, the final part is the item name, and the other parts are the module path.

