# Part 12: Math



## Vector algebra

* Vectors, quaternions, matrices
* Types defined in `types.h`
    * `LocalTransform` is used instead of `Matrix4x4` to set transform on objects
    * In `Matrix4x4` -- scale and rotation is linked, can lead to feedback loops
* Operations defined in `vector3.h`, `matrix4x4.h`, etc



## Random numbers

* LCG -- deterministic sequence
* `random.h`
    * Global random object -- initialized with randomness
    * Systems can create their own local random objects for determinism



## SIMD

* Goal: Abstract over platform specific SIMD instructions
* Problem: APIs do not match exactly
    * Lowest common denominator
    * Still better than floating point
* `float4.h`
* Used to speed up floating point computations (particle system, etc)
    * Some uses suffer from `Vector3` in `float4` antipattern
    * Rewrite could speed up



## Expression languages and compilers

* A need for evaluating mathematical expressions
    * 3 * sin(t) + cos(2*t)
* Converted to "stack machine bytecode" (reverse polish notation)
    * `3 t sin * 2 t * cos +`
* `expression_language.cpp`



## "Vector language"

* Most general way of defining a particle system is to write code
* But we can't compile code dynamically
* Byte code interpretation is slow
* Why?
    * Instruction decoding
    * Branch mispredict, instruction cache miss
* Solution -- parallel byte code (128 objects at a time)
    * Slowness is amortized
* `vector_language.h`
    * Eventually we want to replace all particle simulators with this
* Maybe runtime compile & link should be investigated as an alternative?
