# Conditional Compilation

(TBD)

This section will discuss extensions to WGSL
to support conditional compilation, where the conditions
are set by the host code runtime or build time.

* `#if #else #endif` (or perhaps `#ifdef`) in the style of a C preprocessor 
  is the approach that comes first to mind.
* conditional compilation runs after simple templating but before import processing..
  * (note that we may want to disable code that may not be correct so parsing first is iffy)
* best to define default values in the wgsl for all conditionals
  * IDE tools can't assume any definitions that aren't in the wgsl.
* alternately, consider some kind of [cfg] on elements instead? 