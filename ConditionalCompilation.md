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
  * More structured: ties to source elements not lines
    * Perhaps some optimization potential..
      * For example could parse/lower certain functions ahead of 
        time and easily remove them from the IR vs #if #else which operates on a source level.
    * Is it an wgsl attribute? syntax makes sense but wgsl 
* what are some examples of things to conditionally compile?
  * function variants e.g. select which function variant to use based on condition
  * struct fields e.g. include fields only if condition is set
  * import statements? e.g. import util/fast vs util/regular based on condition
