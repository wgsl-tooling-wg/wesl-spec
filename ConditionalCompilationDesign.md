> Corresponding spec: [ConditionalCompilation.md](ConditionalCompilation.md)

## Initial reflexions

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
* how do we support build time vs runtime linking?
  * e.g. some projects will want prebuilt versions for various mobile configurations.
    * ref [unity shader conditionals](https://docs.unity3d.com/Manual/shader-conditionals.html)
  * should the wesl code distinguish conditions that are settable at compile time from those that are settable at runtime?

## Structured VS. Unstructured

Conditional compilation is a mechanism to modify the source code based on parameters passed to the compiler. We distinguish two kinds:

 * **unstructured**: arbitrary code sections can be injected or conditionaly included. This is what the C preprocessor and macros do, and a lesser version of that is proposed in [`simple templating`](SimpleTemplating.md). 
 * **structured**: only structural elements of the syntax can be manipulated, e.g. a whole declaration, a member, etc. Rust uses this approach with the `#[cfg]` attribute.

We think that *structured* is the way to go. It leads to clearer and safer code, which is more important than implementation complexity in our eyes.
Also, a language that has a good expressive power already should not need a way to hack around with arbitrary code injection, like C macros do.

| unstructured | structured |
|--------------|------------|
| (+) much easier to implement: just look for the `#word` symbols | (-) harder to implement: requires a full parsing |
| (+) often language-agnostic (see the [C preprocessor](https://en.wikipedia.org/wiki/C_preprocessor). Familiar to C developers. | (-) a new syntax needs to be taught, not always self-explanatory |
| (-) poorly integrated in the language, harder to read by humans | (+) is a well-designed syntax feature of the language |
| (+) technically more expressive (e.g. [manipulating identifiers](https://en.wikipedia.org/wiki/C_preprocessor#Token_concatenation)) | (-) can only conditionaly include parts of the syntax tree, poor text generation capabilities. |
| (-) behaves unpredictably, intent and behavior is hidden | (+) intent and behavior is visible at the usage site |
| (-) no type checking, no syntax checking, because code is generated dynamically | (+) can statically check that conditional variants lead to syntactically correct and type-valid code |
| (-) poor IDE support | (+) IDE can check all possible code paths |

## Why do we prefer structured?

TODO

## Syntax extension: using WGSL attributes

We propose to leverage the [*attribute* syntax](https://www.w3.org/TR/WGSL/#attributes) to decorate syntax nodes with conditional compilation attributes.

TODO

## References

[Wikipedia: "Conditional compilation"](https://en.wikipedia.org/wiki/Conditional_compilation)

[C preprocessor](https://en.wikipedia.org/wiki/C_preprocessor)

[Rust conditional compilation](https://doc.rust-lang.org/reference/conditional-compilation.html)

attribute-based conditional compilation in C: ["Conditional Compilation is Dead, Long Live Conditional Compilation!"](https://www.paulgazzillo.com/papers/icse19nier.pdf)[^1]

[^1]: P. Gazzillo and S. Wei, "Conditional Compilation is Dead, Long Live Conditional Compilation!," 2019 IEEE/ACM 41st International Conference on Software Engineering: New Ideas and Emerging Results (ICSE-NIER), Montreal, QC, Canada, 2019, pp. 105-108, doi: 10.1109/ICSE-NIER.2019.00035.
