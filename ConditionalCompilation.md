# Conditional Compilation

This section will discuss extensions to WGSL 
to support conditional compilation based on runtime or build time variables.

(TBD)

* #if / #ifdef on lines is probably the default choice
* conditional compilation runs after variable substitution but before import processing..
* note that we may want to disable code that may not be correct
* define default settings in the wgsl? 
IDE tools can't assume any definitions that aren't in the wgsl.
* alternately, consider some kind of [cfg] on elements instead of text preprocessing?