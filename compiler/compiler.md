# Notes for the Moroso compiler #

This file describes requirements for the compiler for Moroso. To see notes about the programming language it will compile, see [this document](language.md).

The compiler will be written in Rust. It will not be self-hosting.

* The compiler should be able to take some advantage of the special features of the CPU. This includes:
  * Predicated instructions. The compiler should be able to do this automatically.
  * SIMD. We *won't* do auto-vectorization, but will need to support whatever language features allow for vectorized code.

We'll need to support various special language features:
* We need a type inference engine.
