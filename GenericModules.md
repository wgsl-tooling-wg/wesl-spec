# Generics Modules for WGSL

# Summary

We propose adding a mechanism to allow generic programming using WESL modules as an extension to the language.

## Assumptions

Assumes that [`load`](./Imports.md), [Modules](./Modules.md) and [Module Interfaces](./ModulesInterfaces.md) have already been implemented. 

# Motivation

Generic programming is useful for WESL, particularly for libraries.

With generic modules, operations like `Reduce` or `PrefixSum` wouldn't need to be manually rewritten for each combination of element type and binary operation.

Larger projects like GPU accelerated particle systems could also benefit from generic modules. This is because the 
encapsulation of types, functions and other globals in modules provides a means of structuring libraries in a way that is 
readily user extensible, reusable, and maintainable. 


# Guide-level explanation

Generic modules and signatures are declared using the `mod` keyword and like generic types and built-in functions, use angle brackets (`<>`) to denote the generic parameters. Parameters are permitted to be other modules. 

An insantiation of a generic module can be declared inline when used, added to the current namespace using [`include`](./Include.md) (if implemented) or aliased to give the module a concrete name. 

In addition to generic modules, this proposal also requires that [module signatures](./ModulesInterfaces.md) support generic parameters. Module signatures may be used to constrain the type of a generic argument.
Generic module signatures in generic type constraints may additionally use `_` as a "hole" in arguments to indicate that the user doesn't care about the type of a particular generic parameter. 

Below is an annotated example of how generic modules may be used in practise. This has been translated from the 
[StoneBerry WebGPU Repository](https://github.com/stoneberry-webgpu/) into the proposed WESL format

```rescript
// Module signature that simply exposes the single type T. Could perhaps later be sugared to elide the module in follow-up 
// work
mod sig Type {
  type T;
}

// Module signature that exposes a single constant `value`. Could perhaps later be sugared to elide the module in follow-up work
mod sig Const<Type: Type> {
  const value: Type::T;
}

// Abstract representation of a binary operation. 
mod sig BinaryOp<OpElem: Type, LoadElem: Type> {
  // This is a common pattern to allow transfer of type information from generic input to output module
  type LoadElem : LoadElem::T;
  type OpElem : OpElem::T; 
  fn identityOp() -> OpElem;
  fn loadOp(a: LoadElem::T) -> OpElem;
  fn binaryOp(a: OpElem, b: OpElem) -> OpElem;
}

// In future, modules representing numbers, vectors, matrices and other built in types
// would be part of the standard library. But lets define some common operations for now
mod sig Number {
  type T;

  fn add(a: T, b: T) -> T;
  fn identity() -> T;
}

mod Sum<N: Number> {
    struct T {
      sum: N::T;
    }
}


mod SumBinaryOp<N: Number> -> BinaryOp<Sum<N>, Sum<N>> {
    alias OpElem = Sum<N>::T;
    alias LoadElem = Sum<N>::T;

    fn identityOp() -> OpElem {
      return OpElem();
    }
    
    fn loadOp(a: LoadElem) -> OpElem {
        return OpElem(a.sum);
    }

    fn binaryOp(a: OpElem, b: OpElem) -> OpElem {
      return OpElem(N::add(a.sum, b.sum));
    }
}

mod F32 {
  alias T = f32;
  
  fn add(a: T, b: T) -> T {
    return a + b;
  }

  fn identity() -> T {
    return 0.0;
  }
}

mod U32 {
  alias T = u32;
}

// Here we don't care about the exact generic mod values
// passed to BinaryOp as we can extract the underlying types from the module 
// members
mod ReduceWorkgroup<Op: BinaryOp<_, _>, WorkSize: Const<U32>, Threads: Const<U32>> {
    var <workgroup> work: array<Op::OpElem::T, WorkSize::value>; 
    fn reduceWorkgroup(localId: u32) {
        let workDex = localId << 1u;
        for (var step = 1u; step < Threads::value; step <<= 1u) {
            workgroupBarrier();
            if localId % step == 0u {
                work[workDex] = Op::binaryOp(work[workDex], work[workDex + step]);
            }
        }
    }
}

// Same here
mod ReduceBuffer<Op: BinaryOp<_, _>, BlockArea: Const<U32>, WorkSize: Const<U32>, Threads: Const<U32>> {
  // Including brings the module members into the namespace
  include ReduceWorkgroup<Op, WorkSize, Threads>;

  alias Input = Op::LoadElem::T;
  alias Output = Op::OpElem::T;

  struct Uniforms {
      sourceOffset: u32,        // offset in Input elements to start reading in the source
      resultOffset: u32,        // offset in Output elements to start writing in the results
  }

  @group(0) @binding(0) var<uniform> u: Uniforms;                     
  @group(0) @binding(1) var<storage, read> src: array<Input>; 
  @group(0) @binding(2) var<storage, read_write> out: array<Output>;  
  @group(0) @binding(11) var<storage, read_write> debug: array<f32>; // buffer to hold debug values

  override workgroupThreads = 4u;                          

  var <workgroup> work: array<Output, workgroupThreads>; 

  // reduce a buffer of values to a single value, returned as the last element of the out array
  // 
  // each dispatch does two reductions:
  //    . each invocation reduces from a src buffer to the workgroup buffer
  //    . one invocation per workgroup reduces from the workgroup buffer to the out buffer
  // the driver issues multiple dispatches until the output is 1 element long
  //    (subsequent passes uses the output of the previous pass as the src)
  // the same output buffer can be used as input and output in subsequent passes
  //    . start and end indices in the uniforms indicate input and output positions in the buffer
  // 
  @compute
  @workgroup_size(workgroupThreads, 1, 1) 
  fn main(
      @builtin(global_invocation_id) grid: vec3<u32>,    // coords in the global compute grid
      @builtin(local_invocation_index) localIndex: u32,  // index inside the this workgroup
      @builtin(num_workgroups) numWorkgroups: vec3<u32>, // number of workgroups in this dispatch
      @builtin(workgroup_id) workgroupId: vec3<u32>      // workgroup id in the dispatch
  ) {
      reduceBufferToWork(grid.xy, localIndex);
      let outDex = workgroupId.x + u.resultOffset;
      reduceWorkgroup(localIndex);
      if localIndex == 0u {
          out[outDex] = work[0];
      }
  }

  fn reduceBufferToWork(grid: vec2<u32>, localId: u32) {
      var values = fetchSrcBuffer(grid.x);
      var v = reduceSrcBlock(values);
      work[localId] = v;
  }

  fn fetchSrcBuffer(gridX: u32) -> array<Output, BlockArea::value> {
      let start = u.sourceOffset + (gridX * BlockArea::value);
      let end = arrayLength(&src);
      var a = array<Output, BlockArea::value>();
      for (var i = 0u; i < BlockArea::value; i = i + 1u) {
          var idx = i + start;
          if idx < end {
              a[i] = Op::loadOp(src[idx]);
          } else {
              a[i] = Op::identityOp();
          }
      }
      return a;
  }

  fn reduceSrcBlock(a: array<Output, BlockArea::value>) -> Output {
      var v = a[0];
      for (var i = 1u; i < BlockArea::value; i = i + 1u) {
          v = Op::binaryOp(v, a[i]);
      }
      return v;
  }
}
// To actually realize a concrete ReduceBuffer module, we need concrete const values:

mod BlockArea -> Const<U32> {
  const value: u32 = 4u;
}

mod WorkSize -> Const<U32> {
  const value: u32 = 18u;
}

mod Threads -> Const<U32> {
  const value: u32 = 10u;
}

// Putting everything together and into the global namespace
include ReduceBuffer<SumBinaryOp<F32>, BlockArea, WorkSize, Threads>;
```


# Reference-level explanation

Generic modules and signatures are parsed as follows, with spaces and comments allowed between tokens:

```bnf
module_decl:  
  attribute * 'mod' ident _disambiguate_template module_template_param_list ('->'  module_type_specifier_set)? "{" module_member_decl * "}" ";"?
;

module_template_param_list :
  _template_args_start module_template_param_comma_list _template_args_end
;

module_template_param_comma_list : 
  module_template_param ( ',' module_template_param ) * ',' ?
;

module_type_specifier_set :  
  module_type_specifier ('+' module_type_specifier) * 
;

module_template_param : 
  attribute * ident ':' module_type_specifier_set
;

module_sig_decl :
  attribute * 'mod' 'sig' ident _disambiguate_template module_template_param_list '{' global_sig * '}' ';'?
;

module_type_specifier : 
  (module_path '::')? ident _disambiguate_template template_list ?
;

module_path :
  module_path_part ('::' module_path_part)*
;

module_path_part :
  ident _disambiguate_template template_list ?

nested_mod_sig : 
  attribute * 'mod' ident ':' (module_type_specifier ('+' module_type_specifier) *)  ';'
;
```

Where `ident`, `_disambiguate_template`, `template_list`,  `_template_args_start` and `_template_args_end` are defined in the WGSL grammar. 

## Linking WESL Files

Further linking complexity is introduced with this proposal. This is because generic modules need to an additional specialization pass prior to linking.

A basic specialization pass could be implemented according to the following handwavy logic (though includes makes this actually quite a bit subtler, see below for further discussion): 

- for each generic module:
  - for each unique set of generic parameters:
    0. a new concrete module should be produced and given a unique name
    0. for each usage:
      - the generic reference should be rewritten to refer to the concrete module

### Includes

The `include` feature of course breaks this relatively simple scheme because of variables. 

Each time a module with variables is included (regardless of whether it is generic or not), it needs to copy 
the variables into the including namespace, along with at minimum all the dependancies of these variables. 

A naïve approach would be rather to avoid specialization with includes and instead just blindly copy symbols.
This would work, though would lead to larger amounts of output shader code.

### Optimizations

Generics have a lot of room for optimization. 

For example implementations may want to find functions within a module which do not in their usage graph reference any variables or generic parameters; these functions would not need to be specialized. 

<!-- Q: Does this paragraph make sense even? -->
Another potential optimization approach would be producing groups of module members which only depend upon subsets of the declared generic parameters. For each of these sets of module members, specialization would only need to occur for each unique combination of generic argument in the subset.

## Type Checking

Type checking of generics brings its own challenges. The main additions are: module constraints in generic arguments (which might themselves be generic signatures), and the addition of generic "holes" which may be present in module type constraints. 

From a type checking perspective, the constraints is similar to checking modules against their signatures in [Module Interfaces](./ModulesInterfaces.md), however the holes introduce "don't care" semantics. These "don't cares" are necessary 
for brevity but also introduce an indeterminate state to module signatures, relaxing the generic paramaters within a generic module signature to their most general type. 


# Rationale and alternatives

- Why is this design the best in the space of possible designs?
  - Well conceived, consistent abstraction inspired by Ocaml
  - Doesn't introduce introduce additional runtime costs to WESL, and module contents remain valid WGSL
  - Builds upon modules, meaning that there are no additional language concepts to learn beyond understanding modules
  - More flexible than just generic functions
  - Good base for adding syntactic sugar
- What other designs have been considered and what is the rationale for not choosing them?
  - See below
- What is the impact of not doing this?
  - Harder to build abstractions in WESL 
  - Users resorting once again to the escape hatch of basic string templating 


## Alternatives 

### Generic functions
- Less powerful than generic modules
- Harder to build larger, more complicated abstractions
- Potentially simpler to understand for users
- Could be quite easily added later as a syntactic transform using generic modules

### String templating/Substituition
- Hard to have reasonable language server behaviour
- Simple to understand
- Extremely flexible & powerful
- Simple to implement
- Hard to document

### Abstract Modules
_Modules which are partially implemented and require concrete implementation to realize the module_

- Less explicit, and expressive
- Harder to compose behaviour
- Simpler to typecheck
- Likely simpler to implement
- Could be emulated using `include` and module signatures 

## Drawbacks

- Generic modules could be quite complex to implement as soon as you step beyond a naïve approach.
- Unfamiliar programming model
- Without additional syntactic sugar, can sometimes be verbose
- Some form of standard library may become necessary to make using built-in types and functions feasible using this approach

# Future work

## Generic Functions
The implementation of generic functions could depend on generic modules. It would simply be syntactic sugar. 
A generic function could in implementations create a hidden generic module with the same template/generic signature. 

Usages of the generic function would perform a rewrite of the symbol name to refer to the function inside this module
This way a lot of implementation effort could be saved.

For example we could rewrite this generic function 
`fn foo<N: Number>(a: vec4<N::T>, b: vec4<N::T>) -> N {}` as:
```rescript
mod foo<N: Number> {
    fn impl(a: vec4<N::T>, b: vec4<N::T>) -> N::T {}
}
```
 
Usage of this function could then be rewritten from `foo<F32>(vec4<f32>(1.0),  vec4<f32>(2.0))` to `foo<F32>::impl(vec4<f32>(1.0),  vec4<f32>(2.0))`.

Generic functions and modules could take in both modules and concrete functions as arguments, along with constant expressions. Module signatures for generic functions could take the following general pattern:

```rescript
mod sig Fn2<Arg1: Type, Arg2: Type, Returns: Type> {
  fn invoke(arg1: Arg1::T, arg2: Arg2) -> Returns::T;
}

mod sig UnitFn2<Arg1, Arg2> {
  fn invoke(arg1: Arg1::T, arg2: Arg2);
}
```

## Constants

Similarly, the implementation of generic constant parameters could depend on generic modules. It again would simply be syntactic sugar for a generic module. 

For example we could instantiate the generic module `Foo` seen below, as `Foo<10u>` by rewriting `10u` as a module containing a constant value: 

```rescript
mod sig Const<Type: Type> {
  const value: Type::T;
}

mod Foo<V: Constant<U32>> {

}
```

## Types

Finally, the implementation of generic type parameters could depend on generic modules. It once _again_ would simply be syntactic sugar for a generic module. 

We could do this by transforming types provided as an argument to a generic module as an implementation of the `Type`
signature:

```rescript
mod sig Type {
  type T;
}
```

Then the struct `Foo` provided as a generic argument could be rewritten as 

```rescript
mod FooType {
  alias T = Foo;
}
```

## Anonymous Module Declarations
To prevent a prolifieration of single use modules, an additional feature could be added, which is the ability to declare anonymous modules as part of generic arguments. For example this would allow one to write something like this:

```rescript
mod GenericExample<Settings: {
  const frac: f32;
  const height: f32;
}> {
  // Uses settings here
}

alias ConcreteGenericExample = GenericExample<{ 
  const frac = 0.4f; 
  const height = 3f; 
}>;
```
