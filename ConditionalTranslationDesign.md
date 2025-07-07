# Conditional Translation Design

*Also see the spec for [Conditional Translation](ConditionalTranslation.md)*

## Initial reflections

* `#if #else #endif` (or perhaps `#ifdef`) in the style of a C preprocessor
  is the approach that comes first to mind.
* conditional compilation runs after simple templating but before import processing..
  * (note that we may want to disable code that may not be correct so parsing first is iffy)
* best to define default values in the WGSL for all conditionals
  * IDE tools can't assume any definitions that aren't in the WGSL.
* alternately, consider some kind of [cfg] on elements instead?
  * More structured: ties to source elements not lines
    * Perhaps some optimization potential..
      * For example could parse/lower certain functions ahead of
        time and easily remove them from the IR vs #if #else which operates on a source level.
    * Is it an WGSL attribute? syntax makes sense but WGSL
* what are some examples of things to conditionally compile?
  * function variants e.g. select which function variant to use based on condition
  * struct fields e.g. include fields only if condition is set
  * import statements? e.g. import util/fast vs util/regular based on condition
* how do we support build time vs runtime linking?
  * e.g. some projects will want prebuilt versions for various mobile configurations.
    * ref [unity shader conditionals](https://docs.unity3d.com/Manual/shader-conditionals.html)
  * should the WESL code distinguish conditions that are settable at compile time from those that are settable at runtime?

## Structured VS. Unstructured

Conditional compilation is a mechanism to modify the source code based on parameters passed to the compiler. We distinguish two kinds:

* **unstructured**: arbitrary code sections can be injected or conditionally included. This is what the C preprocessor and macros do, and a lesser version of that is proposed in [`simple templating`](SimpleTemplating.md).
* **structured**: only structural elements of the syntax can be manipulated, e.g. a whole declaration, a member, etc. Rust uses this approach with the `#[cfg]` attribute.

We think that *structured* is the way to go. It leads to clearer and safer code, which is more important than implementation complexity in our eyes.
Also, a language that has a good expressive power already should not need a way to hack around with arbitrary code injection, like C macros do.

| unstructured | structured |
|--------------|------------|
| (+) much easier to implement: just look for the `#word` symbols | (-) harder to implement: requires a full parsing |
| (+) often language-agnostic (see the [C preprocessor](https://en.wikipedia.org/wiki/C_preprocessor). Familiar to C developers. | (-) a new syntax needs to be taught, not always self-explanatory |
| (-) poorly integrated in the language, harder to read by humans | (+) is a well-designed syntax feature of the language |
| (+) technically more expressive (e.g. [manipulating identifiers](https://en.wikipedia.org/wiki/C_preprocessor#Token_concatenation)) | (-) can only conditionally include parts of the syntax tree, poor text generation capabilities. |
| (-) behaves unpredictably, intent and behavior is hidden | (+) intent and behavior is visible at the usage site |
| (-) no type checking, no syntax checking, because code is generated dynamically | (+) can statically check that conditional variants lead to syntactically correct and type-valid code |
| (-) poor IDE support | (+) IDE can check all possible code paths |

## Why do we prefer structured?

### Argument 1: structured is sound, and soundness is more important than implementation burden on the long run

Once a couple great and safe linker/compiler implementations for WESL exist, everyone can benefit from them.
Whatever design choice we make now, is potentially a burden in the future. Therefore, we want a robust design.

### Argument 2: structured is better for IDEs and human readers, and is closer to WGSL design choices

`#ifdef`s are not very readable. They don't match the language syntax style, they don't respect the structure (indentation etc), they are more verbose (require `#endif`) and error-prone.
WGSL takes inspiration from Rust (all its keywords were borrowed from Rust, and some elements of its strong safety guarantees, such as making dangling pointer impossible).
Rust uses structured conditional compilation too, with the `#[cfg()]` attribute, which works very similarly to the `@if` attribute.

Structured is great for IDEs too: with a structured `@if`, one can always generate a unified syntax tree

```wgsl
struct Foo {
  a: f32,
  @if(some_condition)
  b: f32
}
```

would turn into a syntax tree along the lines of `(struct foo (member a) (if some_condition (member b)))`.
This means that at the very least, a linker can always check if the WGSL code is syntactically (not semantically) valid.

Meanwhile with `#ifdef`, one can frequently end up with multiple separate syntax trees. The above example could be turned into a single syntax tree. Meanwhile

```wgsl
struct FooBar { // Which bracket is the closing bracket?
  a: f32,
#ifdef some_condition
#else
  b: f32
#endif
}
```

is very hard to turn into a single syntax tree. Juggling two separate syntax trees in a linker or language server is a whole lot of extra effort, so nobody does it.
This example in particular also breaks one IDE feature: Jump-to-matching-bracket

### Argument 3: structured is as expressive as unstructured in real-world use-cases

`@if` is equally powerful as `#ifdef`. Proof:

```text
Assume all combinations of conditions are valid
Expand the #ifdef into all of the combinations
Encode each combination in a separate @if
```

This proof uses code duplication. But in real-world scenarios, one can get around this by decorating with `@if`s only smallest element possible (aka. a struct member instead of the whole struct).

## Syntax extension: using WGSL attributes

We propose to leverage the [*attribute* syntax](https://www.w3.org/TR/WGSL/#attributes) to decorate syntax nodes with conditional compilation attributes.

## References

[Wikipedia: "Conditional compilation"](https://en.wikipedia.org/wiki/Conditional_compilation)

[C preprocessor](https://en.wikipedia.org/wiki/C_preprocessor)

[Rust conditional compilation](https://doc.rust-lang.org/reference/conditional-compilation.html)

attribute-based conditional compilation in C: ["Conditional Compilation is Dead, Long Live Conditional Compilation!"](https://www.paulgazzillo.com/papers/icse19nier.pdf)[^1]

[^1]: P. Gazzillo and S. Wei, "Conditional Compilation is Dead, Long Live Conditional Compilation!," 2019 IEEE/ACM 41st International Conference on Software Engineering: New Ideas and Emerging Results (ICSE-NIER), Montreal, QC, Canada, 2019, pp. 105-108, doi: 10.1109/ICSE-NIER.2019.00035.
