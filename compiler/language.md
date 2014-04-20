# Notes for the Moroso programming language #

The programming language will be inspired by Rust, but won't have some of the parts of it that are more difficult. Some of the features we'll keep include:
* A strong type system
* Namespaces
* Type inference (but probably not as powerful as Rust's)
* Templates

It will not include:
* Rust's memory model. In particular:
  * There won't be multiple pointer types, or concept of ownership.
  * There won't be guarantees about safety.
    * Basically, pointers will be pretty much like C pointers.

In addition, the language should have some way to specify vectors, whose operations will compile down to code that takes advantage of the CPU's SIMD capabilities.
