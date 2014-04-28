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

# Details #
## Basics ##
Function and variable declarations are similar to Rust's:
```
fn frobnicate(x: int, y: int) -> int {
  let z = x+y;
  z+1
}
```
Like Rust, `return` statements are not required, but can be present.
The last expression in a function is its return value.

All variables are mutable by default. They may be declared constant using
the `const` qualifier:
```
  let const z = 5;
  z = 6; // Error!
```
Other qualifiers are expressed in the same way, between the
`let` and the variable name; order doesn't matter. (So they're very
much like qualifiers in C.)

Qualifiers are: `const`, to declare the variable as constant; `static`, to
indicate a variable should not be allocate on the stack and should preserve
its value between calls of the function; and `volatile`, as in C.

## Types ##
The language is strongly typed. Types can be specified explcitly when
declaring a variable:
```
  let z: i8 = 5;
```
or may be inferred:
```
  let x: i8 = 5;
  let z = x; // type of z is inferred as i8.
```

There is no implicit casting. Explicit casts may be done with the `as`
operator:
```
  let x: u8 = 5;
  let y: i8 = x as uint; // Ok: explicit cast.
  let z: i8 = x; // Type error.
```

The language has rust's sized unsigned integer types `u8`, `u16`, `u32` and
signed types `i8`, `i16`, `i32`. It does not have an `int` or `uint` type.
There are no floating point types. At some point support for fixed-point
types may be added, but they will not initially be present.

There is a `bool` type, which may contain only the values `true` and `false`.

Pointer types are like those in C, and the language does not have different
kinds of pointers. Here is a declaration of an uninitialized pointer to a u8:
```
  let x: *u8;
```

Compound types are similar to those in Rust. There are structs:
```
struct Point {
  x: u8,
  y: u8,
}
```
whose members are accessed just like in C or Rust.

There are struct initializers like in Rust:
```
let p: Point = Point { x: 5, y: 6 };
let x: u8 = p.x;
```
Unlike Rust, but like C, there is no "implementation"; that is, you cannot
write something like
```
let p: Point = Point { x: 5, y: 6 };
p.do_stuff();
```
Instead, you'd have to do something like
```
let p: Point = Point { x: 5, y: 6 };
do_stuff(&p);
```

There are enumeration types, like Rust's, which are really sum types:
```
enum Shape {
  Nothing,
  Point(u32, u32),
  Line(u32, u32, u32, u32),
}
```

These can be matched on, but in a much more limited way than Rust:
```
match s {
  Nothing => { return 0; },
  Point(x, y) => { return x+y; },
  Line(x1, y1, x2, y2) => { return x1+y1+x2+y2; },
}
```
Matches must be exhaustive, and cannot contain constants; i.e. we cannot
have a term like
```
  Point(0, y) => { ... },
  Point(1, y) => { ... },
```

There is limited support for templates, which are basically typechecked
pointers. We can declare a templated function:

```
fn linked_list_add<T>(list: *linkedlist<T>, elem: *T) {
  ...
}

```
Any reference to the parameter `T` must be in a pointer. This means that
instantiating the function with multiple different types does not generate
any more code than instantiating it with one type, because all pointer
types are the same size. The only advantage of templates is the type safety
they give to pointers.

When calling a templated function, we can specify the type explicitly
or let it be inferred:
```
let l: *linkedlist<int>;
let i: *int;
...
linked_list_add::<int>(l, i); // Type is specified explcitly.
linked_list_add(l, i); // Also valid: type is inferred.
```

There are function types, written with a `->`:
```
fn apply_to_u8(f: u8 -> u8, x: u8) -> u8 {
  f(x)
}
```
There are no anonymous functions or closures.

Array types are as in C:
```
let x: u8[5] = {1, 2, 3, 4, 5};
```
is an array of five `u8`s. The size is considered part of the array type;
variable-length arrays must be done with pointers. Multi-dimensional
arrays are also as in C.

## Pointers ##
Pointers behave almost exactly like pointers in C. They are nullable, and
have no safety. `&` is used to take the address of a variable; `*` is
used to dereference a pointer. Pointers may be indexed as arrays:
```
let i: i8 = 5;
let pt: *u8 = &i;
*pt = 5;
pt[3] = 5;
```
(TODO: think about the types of indexes. If indexing works the same way it
does in C, this may need a little bit of thought.)

## Control structures ##
Control structures are as in C. C-style `for`, `while`, and `do {...} while`
loops are available. Labels and `goto` statements are not; neither are
`switch` statements (though they may be added later).

## Imports and namespaces ##
TODO.