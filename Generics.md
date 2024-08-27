# Simple Generics for WGSL

(TBD)

Generic programming is useful for wgsl, particularly for libraries.
With generics, wgsl functions like `reduce` or `prefixSum`
don’t need to be manually rewritten for each combination of element type and binary operation.
I’m hoping we can find a fairly minimal design for generics
that is easy for programmers to learn and supportable with modest effort in wesl tools.

To ease implementation effort, I imagine we’ll want to avoid type inference
or type constraints on generics. But without type inference, 
specifying generic types at every call site to a generic function gets verbose and tedious. To avoid that verbosity, let’s allow generic variables on import statements (glslify and wgsl-linker did this too).

I thought we might start by allowing generics only on functions.
We’ll want a design that’s extensible to more features (e.g. generic structs)
of course, 
but we can start with a minimal implementation and add features as they prove necessary.

## Summary

* angle bracket syntax for generic variable declaration, and generic value specification.
* declare generic variables on function declarations, e.g.: `fn foo<E>(arg: E) -> E { let e:E = arg; return e; }`
* within an fn with a generic declaration,
  generic variables names can be used in place of a wgsl type in both the fn declaration and fn body,
  or in a function call expression inside the fn body.
  The generic variable text will be replaced by the generic value text during linking.
  So if `E` is `f32`  the linked wgsl for foo would be: `fn foo(arg: f32) { let e:f32 = arg; return e; }`.
* Note that a linker will generate multiple copies of fn foo() in wgsl,
  one for each unique set of generic arguments. So each fn will have a unique name.
* generic variable values are supplied on import statements or call statements.

      * import with a generic:
        ```
        import util/foo<f32> as foo32;
        main() { foo32(1.0); }
        ```
      * or, call with a generic:
        ```
        foo<f32>(1.0);
        ```

* generic values supplied with imports are single world tokens (typically wgsl type names or function names), 
  or generic variables declared on that function.

## Examples


Simple Example:

* ```
  ./util.wgsl:

  @export fn workgroupMin<E>(elems: array<E, 4>) -> E { }  // E is a generic parameter
  ```

* ```
  ./main.wgsl:

  import ./util/workgroupMin<f32> as workMin; // substitutes f32 for E

  fn main() {
    workMin(a1);  // no generic variables required at the call site 
    workMin(a2);
  }
  ```

Here’s a more complicated case. reduce is parameterized by an element type (e.g. u32) and a binary operation, e.g. max()

* ```
    ./util.wgsl:

    export<E, BinOp> 
    fn reduce(elems: array<E, 2>) -> E { 
      return BinOp(elems[0], elems[1]); 
    }

* ```
    ./main.wgsl:

    import ./ops/binOpMax<f32> as binOpMax;
    import ./util/reduce<f32, binOpMax> as maxF32; 

    fn main() {
      maxF32(a1);
    }
    ```

Note that you can import a generic function w/o providing parameters:

* ```
    ./util.wgsl:

    import binOpMax from ./ops;   // no generic variable specified yet

    export fn reduceMax<E>(elems: array<E, 2>) -> E { 
      return binOpMax<E>(elems[0], elems[1]); // generic value applied at call site
    }
    ```

Re-exporting generics is allowed (presuming we allow re-exporting in general, see [Visibility](./Visiblity.md)):

* ```
    ./lib.wgsl:
    export reduce from util/reduce.wgsl; // re-export at package root level

    ./util/reduce.wgsl:
    export<E, BinOp> fn reduce(elems: array<E, 2>) -> E { }
    ```

## Questions and possible extensions

* Do angle brackets conflict or comport with wgsl templates?
* Can you export a generic function after variable substitution too? or only the generic version
* Allow generic values to be pulled from runtime parameters?
  wgsl-linker currently recognizes an `ext.` prefix to get variable values from the runtime caller.
  e.g. ext.workgroupSize would patch in runtime variables at link time.
  Hopefully we can address that with runtime #define, we’ll see.
* Currently there are no type constraints available for generic variable declarations..
  Simply substituting parameters and letting dawn or naga typecheck at runtime seems ok for now.
  A future type checker could check annotation uses are valid by substituting generic parameters
  and type checking the expanded wgsl.
  And of course a future version of wgsl or wesl generics could add explicit type constraints on generic variables.

* Generics on structs too?

  * ```
      ./util.wgsl:
      export struct Point<T> { position: vec2<T>, color vec3f } 

      ./main.wgsl
      import Point<u32> as UPoint from ./util

      fn main() {
        let p = UPoint(vec2u(0, 0), vec3f(.5, .5, .5));
      }
      ```

* The reduce example makes me think whether we could call a generic function recursively,
  and what would that do.
  In theory, the following would unroll to N nested function calls.
  The wgsl compilers may be good at flattening this.

* ```
    fn accumulate<E, Op, N>(acc: E, elems: array<E, N>) -> E {
      if N >= 2 {
        return accumulate<E, Op, N-1>(accumulateBinOp(E, elems[N-1]), elems);
      else {
        return acc;
      }
    }

    fn op_add<E>(e1: E, e2: E) {
      return e1 + e2;
    }

    fn array_sum<E, N>(elems: array<E, N>) -> E {
      return accumulate<E, op_add<E>, N>(0, elems);
    }
  ```

  Array_sum has a nested generic! This is cool.
  
  Also, some SFINAE I guess: because 0 is AbstractInt, it can subtitute E with u32 or i32, BUT not f32 afaik, because 0 is not AbstractFloat. This is somewhat disappointing. 
