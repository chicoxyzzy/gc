# GC v1 Extensions

*Note: This design is still in flux!*

See [overview](Overview.md) for background.


## Language

Based on the following proposals:

* [reference types](https://github.com/WebAssembly/reference-types), which introduces type `anyref` etc.

* [typed function references](https://github.com/WebAssembly/function-references), which introduces types `(ref $t)` and `(optref $t)` etc.

Both these proposals are prerequisites.


### Types

#### Value Types

* `eqref` is a new reference type
  - `reftype ::= ... | eqref`

* `i31ref` is a new reference type
  - `reftype ::= ... | i31ref`

* `rtt <typeuse>` is a new reference type that is a runtime representation of type `<typeuse>` (see [Runtime types](#runtime-types))
  - `reftype ::= ... | rtt <typeuse>`
  - `rtt t ok` iff `t ok`

* Note: types `anyref` and `funcref` already exist via [reference types proposal](https://github.com/WebAssembly/reference-types), and `ref $t` and `optref $t` via [typed references](https://github.com/WebAssembly/function-references)


#### Type Definitions

* `deftype` is a new category of types that generalises the existing type definitions in the type section
  - `deftype ::= <functype> | <structtype> | <arraytype>`
  - `module ::= {..., types vec(<deftype>)}`

* `structtype` describes a structure with statically indexed fields
  - `structtype ::= struct <fieldtype>*`

* `arraytype` describes an array with dynamically indexed fields
  - `arraytype ::= array <fieldtype>`

* `fieldtype` describes a struct or array field and whether it is mutable
  - `fieldtype ::= <mutability> <storagetype>`
  - `storagetype ::= <valtype> | <packedtype>`
  - `packedtype ::= i8 | i16`

* Unpacking a storage type yields `i32` for packed types, otherwise the type itself
  - `unpacked(t) = t`
  - `unpacked(pt) = i32`


#### Type Uses

A *type use* denotes a user-defined or pre-defined data type:

* `typeuse ::= <typeidx> | any | func | eq | i31 | rtt`

* In the binary encoding,
  - the `<typeidx>` is encoded as a (positive) signed LEB
  - the others use the same (negative) opcodes as `anyref`, `funcref`, `eqref`, `i31ref`, `rtt`, respectively


#### Imports

* `type <typetype>` is an import description with an upper bound
  - `importdesc ::= ... | type <typetype>`
  - Note: `type` may get additional parameters in the future

* `typetype` describes the type of a type import, and is either an upper bound or a type equivalence
  - `typetype ::= sub <typeuse> | eq <typeuse>`

* Type imports have indices prepended to the type index space, similar to other imports.
  - Note: due to bounds, type imports can be mutually recursive with other type imports as well as regular type definitions. Hence they have to be validated together with the type section.


#### Exports

* `type <typeidx>` is an export description
  - `exportdesc ::= ... | type <typeidx>`
  - `type $t ok` iff `$t` is defined in the context


#### Subtyping

Greatest fixpoint (co-inductive interpretation) of the given rules (implying reflexivity and transitivity).

In addition to the rules for [basic](https://github.com/WebAssembly/reference-types/proposals/reference-types/Overview.md#subtyping), and [typed](https://github.com/WebAssembly/function-references/proposals/function-references/Overview.md#subtyping) reference types:

* `eqref` is a subtype of `anyref`
  - `eqref <: anyref`
  - Note: `i31ref` and `funcref` are *not* a subtypes of `eqref`, i.e., those types do not expose reference equality

* `nullref` is a subtype of `eqref`
  - `nullref <: eqref`

* `i31ref` is a subtype of `anyref`
  - `i31ref <: anyref`
  - Note: `i31ref` is *not* a supertype of `nullref`, i.e., nut nullable

* Any optional reference type (and thereby respective concrete reference type) is a subtype of `eqref` if its not a function
  - `optref $t <: eqref`
     - if `$t = <structtype>` or `$t = <arraytype>`
     - or `$t = type rt` and `rt <: eqref`
  - TODO: provide a way to make data types non-eq, especially immutable ones

* Structure types support width and depth subtyping
  - `struct <fieldtype1>* <fieldtype1'>* <: struct <fieldtype2>*`
    - iff `(<fieldtype1> <: <fieldtype2>)*`

* Array types support depth subtyping
  - `array <fieldtype1> <: array <fieldtype2>`
    - iff `<fieldtype1> <: <fieldtype2>`

* Field types are covariant if they are immutable, invariant otherwise
  - `const <valtype1> <: const <valtype2>`
    - iff `<valtype1> <: <valtype2>`
  - `var <valtype> <: var <valtype>`
  - Note: mutable fields are *not* subtypes of immutable ones, so `const` really means constant, not read-only

* `rtt t` is a subtype of `anyref`
  - `rtt t <: anyref`
  - Note: `rtt t1` is *not* a subtype of `rtt t2`, even if `t1` is a subtype of `t2`; such subtyping would be unsound, since RTTs are used in both co- and contravariant roles (e.g., both when constructing and consuming a reference)


### Runtime

#### Runtime Types

* Runtime types (RTTs) are explicit values representing types at runtime; a value of type `rtt <t>` is a dynamic representative of static type `<t>`.

* All RTTs are explicitly created and all operations involving dynamic type information (like casts) operate on explicit RTT operands.

* There is a runtime subtyping hierarchy on RTTs; creating an RTT requires providing a *parent type* in the form of an existing RTT; the RTT for `anyref` is the root of this hierarchy.

* An RTT t1 is a *sub-RTT* of another RTT t2 iff either of the following holds:
  - t1 and t2 represent the same static type, or
  - t1 has a parent that is a sub-RTT of t2.

* Validation requires that each parent type is a representative of a static supertype of its child; runtime subtyping hence is a sub-relation of static subtyping (a graph with fewer nodes and edges).

* At the same time, runtime subtyping forms a linear hierarchy such that the relation can be checked efficiently using standard implementation techniques (it is a tree-shaped graph).


#### Values

* Creating a structure or array optionally allows supplying a suitable RTT to represent its runtime type; it defaults to `anyref` if none is given.

* Each reference value has an associated runtime type:
  - For structures or arrays, it is the RTT provided upon creation, or `anyref` if none.
  - For `i31ref` references it is the RTT for `i31ref`.
  - For `null` it is the RTT for `nullref`.

* The so-defined runtime type is the only type information that can be discovered about a reference value at runtime; a structure or array with RTT `anyref` thereby is fully opaque to runtime type checks (and an implementation may choose to optimize away its RTT).


### Instructions

#### Equality

* `ref.eq` compares two references whose types support equality
  - `ref.eq : [eqref eqref] -> [i32]`


#### Functions

Perhaps add the following short-hands:

* `ref.is_func` checks whether a reference is a function
  - `ref.is_func : [anyref] -> [i32]`
  - equivalent to `(rtt.funcref) (ref.test anyref funcref)`

* `ref.as_func` converts to a function reference
  - `ref.as_func : [anyref] -> [funcref]`
  - traps if reference is not a function
  - equivalent to `(rtt.funcref) (ref.cast anyref funcref)`


#### Structures

* `struct.new <typeidx>` allocates a structure of type `$t` and initialises its fields with given values
  - `struct.new $t : [t*] -> [(ref $t)]`
    - iff `$t = struct (mut t)*`
  - equivalent to `struct.new_rtt $t anyref (rtt.anyref)`

* `struct.new_rtt <typeidx> <typeuse>` allocates a structure of type `$t` with RTT information determining its [runtime type](#values) and initialises its fields with given values
  - `struct.new_rtt $t t' : [(rtt t') t*] -> [(ref $t)]`
    - iff `$t = struct (mut t)*`
    - and `ref $t <: t'`

* `struct.new_default <typeidx>` allocates a structure of type `$t` and initialises its fields with default values
  - `struct.new_default $t : [] -> [(ref $t)]`
    - iff `$t = struct (mut t)*`
    - and all `t*` are defaultable

* `struct.get_<sx>? <typeidx> <fieldidx>` reads field `$x` from a structure
  - `struct.get_<sx>? $t i : [(optref $t)] -> [t]`
    - iff `$t = struct (mut1 t1)^i (mut ti) (mut2 t2)*`
    - and `t = unpacked(ti)`
    - and `_<sx>` present iff `t =/= ti`
  - traps on `null`

* `struct.set <typeidx> <fieldidx>` writes field `$x` of a structure
  - `struct.set $t i : [(optref $t) ti] -> []`
    - iff `$t = struct (mut1 t1)^i (var ti) (mut2 t2)*`
    - and `t = unpacked(ti)`
  - traps on `null`


#### Arrays

* `array.new <typeidx>` allocates an array of type `$t` and initialises its fields with a given value
  - `array.new $t : [t i32] -> [(ref $t)]`
    - iff `$t = array (mut t)`
  - equivalent to `array.new_rtt $t anyref (rtt.anyref)`

* `array.new_rtt <typeidx> <typeuse>` allocates a array of type `$t` with RTT information determining its [runtime type](#values)
  - `array.new_rtt $t t' : [(rtt t') t i32] -> [(ref $t)]`
    - iff `$t = array (mut t)`
    - and `ref $t <: t'`

* `array.new_default <typeidx>` allocates an array of type `$t` and initialises its fields with the default value
  - `array.new_default $t : [i32] -> [(ref $t)]`
    - iff `$t = array (mut t)`
    - and `t` is defaultable

* `array.get_<sx>? <typeidx>` reads an element from an array
  - `array.get_<sx>? $t : [(optref $t) i32] -> [t]`
    - iff `$t = array (mut t')`
    - and `t = unpacked(t')`
    - and `_<sx>` present iff `t =/= t'`
  - traps on `null` or if the dynamic index is out of bounds

* `array.set <typeidx>` writes an element to an array
  - `array.set $t : [(optref $t) i32 t] -> []`
    - iff `$t = array (var t')`
    - and `t = unpacked(t')`
  - traps on `null` or if the dynamic index is out of bounds

* `array.len <typeidx>` inquires the length of an array
  - `array.len $t : [(optref $t)] -> [i32]`
    - iff `$t = array (mut t)`
  - traps on `null`


#### Integer References

Tentatively, support a type of guaranteed unboxed scalars.

* `i31.new` creates an `i31ref` from a 32 bit value, truncating high bit
  - `i31.new : [i32] -> [i31ref]`

* `i31.get_u` extracts the value, zero-extending
  - `i31.get_u : [i31ref] -> [i32]`

* `i31.get_s` extracts the value, sign-extending
  - `i31.get_s : [i31ref] -> [i32]`

Perhaps also the following short-hands:

* `ref.is_i31` checks whether a reference is an i31
  - `ref.is_i31 : [anyref] -> [i32]`
  - equivalent to `(rtt.i31ref) (ref.test anyref i31ref)`

* `ref.as_i31` converts to an integer reference
  - `ref.as_i31 : [anyref] -> [i31ref]`
  - traps if reference is not an integer
  - equivalent to `(rtt.i31ref) (ref.cast anyref i31ref)`


#### Runtime Types

* `rtt.anyref` returns the RTT of type `anyref` as a sub-RTT of only itself
  - `rtt.anyref : [] -> [(rtt anyref)]`
  - this is a *constant instruction*

* `rtt.new <typeuse> <typeuse>` returns the RTT of the specified type as a sub-RTT of a given parent RTT operand
  - `rtt.new t t' : [(rtt t')] -> [(rtt t)]`
    - iff `t <: t'`
  - multiple invocations of this instruction with the same operand yield the same observable RTTs
  - this is a *constant instruction*


#### Casts

* `ref.test <typeuse> <typeuse>` tests whether a reference value's [runtime type](#values) is a [runtime subtype](#runtime) of a given RTT
  - `ref.test t t' : [t (rtt t')] -> [i32]`
     - iff `t' <: t <: anyref`
  - returns 1 if the first operand's runtime type is a sub-RTT of the RTT operand, 0 otherwise

* `ref.cast <typeuse> <typeuse>` casts a reference value down to a type given by a RTT representation
  - `ref.cast t t' : [t (rtt t')] -> [t']`
     - iff `t' <: t <: anyref`
  - traps if the first operand's runtime type is not a sub-RTT of the RTT operand

* `br_on_cast <labelidx> <typeuse> <typeuse>` branches if a value can be cast down to a given reference type
  - `br_on_cast $l t t' : [t (rtt t')] -> [t]`
    - iff `t' <: t <: anyref`
    - and `$l : [t']`
  - branches iff the first operand's runtime type is a sub-RTT of the RTT operand
  - passes cast operand along with branch


## Binary Format

TODO.


## JS API

See [GC JS API document](MVP-JS.md) .


## Questions

* Should RTT presence be made explicit in struct types and ref types?
  - for example, `(struct rtt ...)` and `rttref <: anyref`
  - only these types would be castable

* Provide a way to make data types non-eq, especially immutable ones?
