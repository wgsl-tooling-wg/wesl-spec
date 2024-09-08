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
