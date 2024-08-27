# Simple Templating

(TBD)

Several community projects have found the need to modify WGSL at runtime or compile time based on
variables known to the host code.

For example the host code might a the WorkgroupSize for use in WGSL `@workgroup_size()` declarations,
or set the BlockSize for WGSL arrays in an image processing algorithm, or even set the array type
for a generic function.

Typically the the host code passes a record of name-value pairs, and corresponding names
are replaced in the WGSL code.

WGSL supports `override` declarations for a similar purpose, but overrides are not applicable
everywhere in wgsl.

Issues

* Default values should be visible in the WGSL, so that language servers have something to go on
* `#define`?
* special syntax for names to be replaced, e.g. `#{WORKGROUP_SIZE}` or just an all caps convention like C?
* Process as a preprocessor step, before other WESL processing.
* Runtime modifiable variables are visible to the host, and so changes to names affect the api of the wgsl package.
  Can we make the list of names available so that host code can autocomplete and validate names?
* Are values opaque strings
* should we somehow be integrating or extending override?
