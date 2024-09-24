# Struct Inheritance for WGSL

(TBD)

It’s convenient to be able to express structures as combinations of other structures as in Java/TypeScript.

wgsl-linker currently supports an `extends` syntax for struct inheritance (but `extends` is only lightly
used upstream).

Combining structures by simple composition is possible in standard WGSL.
by referencing through a member field. But:

* Using fields of the referenced struct requires an additional field dereference "." in the syntax
  for each level of indirection.
* Some abstractions may be hard to express w/o inheritance (since we also don't have interfaces in WGSL).

Issues:

* TBD demonstrate an example or drop the feature.
* Also, would generics for structs be an alternate way to parameterize composition?
