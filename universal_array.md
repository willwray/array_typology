# Design for a universal C++ array type

<details><summary><b>Contents</b> (folding section, click to unfold)</summary>

* [C++ array Status Quo](#c++-array-status-quo)
* [Introduction](#introduction)
* [Array concepts](#array-concepts)
* [Array reference vs value](#array-reference-vs-value)
* [Storage, layout and index view](#storage,-layout-and-index-view)
* [Multi indexing `(i,j,k)` vs subobject indexing `[i]`](#multi-indexing-(i,j,k)-vs-subobject-indexing-[i])
* [Static vs dynamic](#static-vs-dynamic)
* [Introducing ray](#introducing-ray)
* [Revealing uvray](#revealing-uvray)
* [Aggregate by nature or by nurture](#aggregate-by-nature-or-by-nurture)
* [Array declaration, 1D and 2D intrinsic value type](#array-declaration,-1D-and-2D-intrinsic-value-type)
* [Array value initialization, assignment and copy](#array-value-initialization,-assignment-and-copy)
* [Simple specializations](#simple-specializations)

</details>

## C++ array Status Quo

See [C++ Array Phenomenology and Typology](typology.md) for background.  
C++ inherits its language array type from C, with a few changes.  
The C++ standard library provides a few low-level array-like types:

* `std::array<T,N>` wraps C-array, prevents decay, enables copy
* `std::unique_ptr<T[]>` owns a heap array of runtime length
* `std::span<T,N>` non-owning range of elements of static length
* `std::span<T>` non-owning range of elements of runtime length
* `std::string_view` span of immutable `char`, shrinkable length

Length is either fixed at compile-time or set at runtime, as an upper limit;  
`span` provides subspan and `string_view` supports dynamic shrinkage.  
Runtime growth is considered out-of-scope for low-level array types.  
Of the growable container types, the two most relevant are:

* `std::vector<T>` owning dynamic length
* `std::valarray<T>` owning dynamic length, numeric, RIP

All provide indexing `operator[](int)` characteristic of array.  
(So do random-access iterators, akin to pointers into arrays.)

None are well suited for multi-dimensional data.  
For multi-dim use cases, these are proposed:

* `std::mdarray` a multi-dimensional array sibling of...
* `std::mdspan` a multi-dimensional, non-owning, span

## Introduction

This is a design sketch for a low-level multi-dimensional array class.
It explores an adjacent design space to `mdspan` and `mdarray` std proposals.
Like them, it is primarily targetted at numerical matrix applications
(but works for any non-trivial element type).
It aims to "make simple tasks simple"
while allowing for efficient access patterns.

This design promotes value semantics with deep
assignment and comparison.
All indexing operations return references.
Array references model core language references;
they act as aliases to array values.
Sub-rank indexing returns sub-array.

The focus is on data _layout and indexing_ schemes 'LAI'.
Schemes are specified in terms of values of `bounds` type whose constructors form an EDSL (embedded domain-specific language) for computing offsets from indices based on a set of parameters.
The LAI EDSL is entirely separable from the array type (which can be seen as a LAI delivery vehicle).
Only simple schemes are covered here.

Rectangular matrices can be stored or indexed in permuted orders to support row-major or column-major layout,
transposed-view indexing and tiled block-matrix layouts.
Triangular data can be indexed as upper, lower, symmetric etc.
Sparse data layout, associative or unstructured arrays are all out of scope, for now.
The aim is to fit the known common dense layouts
with flexible indexing.
A particular aim is for control of small static-sized block schemes for SIMD and GPU performance tuning.

The design avoids Policy template parameters.
Instead, the array's handle type can extend the built-in selection of schemes
or be chosen as, say, a `unique_ptr` to inject ownership.
There's no direct support for dynamic memory allocation.

It is currently a design-stage 'strawtype';
an unorthodox, highly experimental, only partly-implemented prototype with no real usage.
The aim is to implement, evaluate and coevolve with usage.
As such, the array type itself is only the start of the story,
the ending to follow if it proves itself in use.

Future work. Short term; coevolve design with usage.
Explore how layout, indexing and arrays fit with concepts and ranges.
Write linear algebra components.
Medium term; coroutines, generators.
Long term; native C++ BLAS-like CLAS library.

## Array concepts

The design presented below covers all kinds of array type.  
They are distinguished by concept rather than by type name.

Each type fits exactly one of these concepts:

* `ival` : an 'intrinsic' value like `std::array<T,N>`
* `umov` : an owning-handle like `std::unique_ptr<T[]>`
* `refs` : a non-owning handle like `const std::span`

(Here, by design, 'extrinsic' handles `umov`, `refs` are immutable  
even if the data referenced by the handle is generally mutable).

(Member functions `.ival()`, `.uval()` and `()` - `operator()` -  
perform intrinsic-copy, unique-clone and conversion to reference.)

The three disjunctions are:

* `v` == `ival` || `umov` : owning value type, intrinsic or extrinsic
* `x` == `umov` || `refs` : handle/view type, owning or non-owning  
* `s` == `refs` || `ival` : non-owning handle or intrinsic value

`v` : value, `x` : extrinsic, `s`: stack / scoped / static (no `new`/`delete`):

```ascii
                  INTRINSIC
                    value
                      i.
                     /  \
                    /ival\
      Value     v: /      \  s: Stack
      == owning   /        \    == no new/delete
                 /umov  refs\
                u.-----------r.
         UNIQUE        x:       REFERENCE
move-only-value -- eXtrinsic -- ref or ptr / view
  owning-handle    == handle    non-owning handle
                   == view
```

Three vertices reflect three polar choices, opposite edges their duals:

* **i**: Intrinsic vs. extrinsic: **x**
* **u**: Unique-ownership (heap) vs. not (stack): **s**
* **r**: Reference vs. value: **v**

'Reference' is for non-owning `refs` vs. 'value' for owning `ival`, `umov`.

Notice that `umov` is a value despite that `unique_ptr` has _ptr suffix  
and also classes as an extrinsic view; crucially it's a move-only view.  
Enforcement of unique ownership is key  
(and not the fact that it does this by management of a pointer).

### Refinement (concept constraint details, WIP)

How does this fit with std concepts and ranges? (WIP)

All array-like types model [`contiguous_range`](https://en.cppreference.com/w/cpp/ranges/contiguous_range)  
by virtue of `begin()`, `end()` and contiguous `data()`.  

Intrinsic `ival`, effectively `contiguous_range_intrinsic`,  
requires either `is_array` or `is_layout_compatible` with a  
struct containing an array.

Extrinsic types are effectively views, let's say `contiguous_view`.  
`umov` is also move-only, effectively `contiguous_view_unique`.  
We could further refine `refs` as pointer-like or reference-like:

* `ptr` : pointer-like, has null representation (c.f. `view::maybe`)
* `ref` : reference-like, no null representation

(As references are immutable by design there is no rebinding;  
null is an immutable state, not an empty state to be bound later.)

A `ref` refinement could be required to refer to an array (`core_ref`?)

Array iterators behave as array references:

* `iter` : array iterator type

## Array reference vs value

Non-owning `refs` reference types are central to the design, as the return type of all indexing expressions.
Owning value types  play a secondary, though important, supporting role
(`mdspan` didn't gain a sibling `mdarray` for many years and proposal iterations).
Array code does most of its work indexing or using view types for indirect data access.
First-class array values are a convenience for dealing with data buffers and ownership.

Concept names `ival`, `umov` vs. `refs` accentuate this 'value' vs. 'reference' divide
based on value ownership rather than on value semantics.
Semantics-wise, in this design _all_ array-like types are given value semantics.
That is, even non-owning references behave like values with assignment and comparison operations
acting _deep_, through the reference, to the underlying referred-to data.
This can be confusing, controversial even, so the rationale is covered here in some detail.




Firstly, value semantics is simply the useful choice.
In the case of a C-array reference-wrapper,
it exists only to add regularity via copy, assignment and comparison operators.

Secondly, core-language references serve as aliases, mostly indistinguishable from their bound value.
There is no re-binding and no null representation.
Once a reference has been initialized it is effectively immutable.

**Reference types** are non-owning and immutable. They are either core-language reference types, `&`/`&&`,
or reference-wrapper proxy classes holding a pointer or reference 'handle' to the element storage and array extent information.
Initialization of a reference type acts like reference-binding, not like initialization of the referred-to value.
Then, post initialization, the reference is immutable, even if it refers to mutable data
(so any dynamic extents must be set at initialization as they are not dynamically mutable).

**Value types** are 'owning'. They are either 'intrinsic' multi-dimensional versions of `std::array`,
aggregate structs containing a built-in array of elements for which extents must be static,
or they are 'extrinsic' owning handles to a `new`'d array, which may have dynamic bounds. Array values may be mutable, or immutable.
Code should be aware that array values are potentially expensive to copy, if they are copyable,
and that `move` may be equivalent to `copy`, or the value may be `move`-only (or even immovable).

## Array Generics vs Introspection

> We need to separate the implementation from the interface  
but not at the cost of totally ignoring complexity.  
...  
Complexity assertions have to be part of the interface.

* Alex Stepanov, [Dr-Dobbs interview by Al Stevens](http://stepanovpapers.com/drdobbs-interview.html), March 1995

There is tension between generic programming vs special casing for performance.
Andrei Alexandrescu promotes Design by Introspection as a way to progress beyond concepts and type constraints.

The refined array concepts allow to pass more type information than 'contiguous' alone.
The disjunctions give interfaces wiggle-room with argument and return type.
Implementations can choose what to do based on priorities that could be encoded as user-defined properties:

```c++
template <array_x A, array_x B>
array_v auto add(A a, B b) requires same_shape<A,B>
```

Here, the implementation can reuse an owning-handle argument as the return type
or return an intrinsic value if the size is below some threshold.

## Storage, layout and index view

For multi-dimensional data, it helps to distinguish these three levels:

* **Storage**: preferably a contiguous 1D sequence of elements.  
Storage is either owned by the array or it is referenced.

* **Layout**: a logical format of the storage as a hierarchy of subranges.  
Usually rectangular, can be triangular or other 'geometry'.

* **Index view**: The shape advertised by the type's indexing API.

The layers are sketched below for four matrix types with `3x2` layout and/or index view:

1. Nested-store row-major `E[3][2]`; C-array 
2. Flat-store row-major `3x2`
3. Flat-store col-major `3x2` (i.e. laid out as `2x3`)
4. Flat-store row-major `3x2` viewed as transposed `3x2` = `2x3`

Roles of storage, layout and indexing
are combined in nested C-arrays;
storage is nested subarray subobjects,
elements are laid out directly into storage
and indexing mirrors layout:

```c++
       3x2 matrix
                       col i:               (row,col)
   2 cols  . .             \  0 1
       . | a b |          0 | a b |       |(0,0) (0,1)|
3 rows . | c d |   row j: 1 | c d | (j,i) |(1,0) (1,1)|
       . | e f |          2 | e f |       |(2,0) (2,1)|

                  Storage        Layout     Index view
    [3][2]      ___ ___ ___    ___ ___ ___    | a b |
Nested storage |?,?|?,?|?,?|  |a,b|c,d|e,f|   | c d |
 (2D C-array)   ‾‾‾ ‾‾‾ ‾‾‾    ‾‾‾ ‾‾‾ ‾‾‾    | e f |

    {3,2}      _____________  _____________   | a b |
 Flat storage  |?,?,?,?,?,?|  |a,b|c,d|e,f|   | c d |
  Row-major    ‾‾‾‾‾‾‾‾‾‾‾‾‾  ‾‾‾‾‾‾‾‾‾‾‾‾‾   | e f |

  swap{3,2}    _____________  _____________   | a b |
 Flat storage  |?,?,?,?,?,?|  |a,c,e|b,d,f|   | c d |
 Column-major  ‾‾‾‾‾‾‾‾‾‾‾‾‾  ‾‾‾‾‾‾‾‾‾‾‾‾‾   | e f |
                                           ------------
  {3,2}swap    _____________  _____________  | a c e |
Flat row-major |?,?,?,?,?,?|  |a,b|c,d|e,f|  | b d f |
Transpose view ‾‾‾‾‾‾‾‾‾‾‾‾‾  ‾‾‾‾‾‾‾‾‾‾‾‾‾
                                        |(0,0) (0,1) (0,2)|
  Swapped index order; (j,i) => (i,j):  |(1,0) (1,1) (1,2)|
```

In the last two cases, roles of storage, layout and indexing are all different,
with strided access between layout and index.

## Multi indexing `(i,j,k)` vs subobject indexing `[i]`

For array classes, indexing is usually done via function calls.
In fact _the_ 'function call' operator is conventionally used for indexing into multi-dimensional data, as it is here.

Here, it takes any number of indices from zero up to the array rank:

* `a(i...)` : multi-indexing, returns a reference-wrapper to a subarray

The familiar indexing operator `[i]` is defined only when the single index `i` addresses an object.
It may be implemented via implicit conversion to an array lvalue or by an `operator[](I)` overload with the same meaning:

* `a[i]` : subobject indexing, returns a reference (`&` or `&&`) to the subobject  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(the index must address an object, otherwise `a[i]` is not defined)

(In core language `a[i]` is an id expression for array element `i`, a subobject, and names the lvalue.
Currently only a structured binding can alias an array element id.
There's no way to deal directly with ids, say to return them from functions.
For now we must deal with references.)

### The syntax: it's fine

Function call syntax for array indexing is surprising at first for some versed in C++;
square brackets shout 'indexing' in C/C++.
Function call syntax implies all the generality of functions.
That's actually appropriate here as data could be distributed with complex schemes for layout and indexing.
For those more versed in array manipulation this is a non-issue.
It is quick to get used to.

P2128 _Multidimensional subscript operator_ proposes a variadic `operator[](I...)` overload taking one or more args.
Indiscriminate use of such an `a[i...]` overload could confuse semantics of multi-dimensional array access
by mixing up multi-indexing with subobject indexing.
It is my opinion that the proposal is poorly motivated, possibly deleterious.

## Static vs dynamic

Strong static typing is the default;
static bounds are retained as far as possible with subarray views keeping a 'breadcrumb' trail back to the full array type.
Even array elements are viewed as subarrays,
returned as proxies from multi-index expressions,
so they have knowledge of the parent array type.

Static bounds are part of the declared type.
Any bounds declared as dynamic placeholders must be provided on construction of the declared type,
to be stored in-class following the array storage handle.
Dynamic bounds may instead be 'baked in'
as external static data of a uniquely-typed class spawned from the declared type.

## Introducing `ray`

They say that naming is the hardest thing; let's call this thing '`ray`', a multi-dimensional array alias:

```c++
template <typename T, bounds... b> using ray = uvray</**/>;
```

The variadic `bounds...` parameter matches a single argument or no argument.
In other words, `bounds...` is variadic only to allow the zero Arg case
(a possible avenue for expansion is to allow more arguments):

```c++
ray<E,b> eb;       // Single bounds argument; b
ray<A> a;          // No bounds arg; empty pack
// ray<E,b,c> ebc; // More than one Arg undefined
```

The `ray` alias does normalization such that different template argument specifications
with the same semantic meaning will resolve to the same type.

The `ray` API is C++20, experimental and unstable;
if it sees usage then the API will be stablized at some point.

## Revealing `uvray`

The underlying value type is called '`uvray`':

```c++
template <typename U, auto... a> struct uvray {/**/};
```

Users should not expose themselves directly to `uvray`
but bask only in the indirect `ray` alias
That is, the `uvray` interface is not user-facing.
It could become a developer API.

The API targets C++17 with possible c++17 implementation.
It may be back portable to C++14
(specific specializations of `auto` argument should mangle the same).
Compatibility with C++17 is a low priority aim.
Compatibility with earlier standards is not an aim.

Minimality is a design goal.
Preferably `uvray` will be implemented as a single specialization with no base classes.

## Aggregate  

All `ray` types are aggregate.
'Intrinsic' array value types, like std::array, are aggregate by nature.
Array reference and owning-handle types traditionally have constructors.
This design makes them aggregate too.

The idea is that array types are simple wrappers of indexing functionality.
An array does not have invariants in and of itself for constructors to enforce.
Non-explicit constructors are a kind of implicit conversion which may weaken the type.
Explicit constructors may as well be free function helpers.

Unlike `std::span`, `ray`'s are not intended to serve as a concrete 'vocabulary' types.
Such interface types have constructors to covert to their common type; a kind of type-erasure.
Instead, `ray` arguments are accepted as-is or rejected by constraints
such as `square`, `symmetric`, `upper_triangular`.
Full type information is retained all the way down to computational kernels.

Lack of constructors pushes responsibility for correct initialization up to the user.
For this, `make_` functions should be provided, e.g. for arrays with resource-holding handles;

```c++
auto ur = make_unique_ray<E,b>(...);
```

Aggregate types simpilfy the design. Extension is possible by injection or by inheritance.

With nurture - usage experience, experimentation and evaluation -
this design may lead to concepts for static-sized arrays and multi-dimensional ranges.

## Array declaration, 1D and 2D intrinsic value

We start with 'intrinsic' array values, that hold the actual array as member data
(reference types and owning-handle value types are covered later).
Dealing with intrinsic values is pedagogical, if not always straightforward.

Recall `ray`'s template alias signature:

```c++
template <typename T, bounds... b> using ray = /**/
```

The `T` argument specifies the array element type `E` (and its storage)
while `bounds...` specifies layout, extents, etc.
If, however, `bounds...` is empty, then the type `T`  must specify the full array type `A`;
element, storage, bounds and all -
it is an error to declare a `ray` 'scalar':

```c++
ray<int> // Compile error
```

A `ray` can be declared as a direct wrapper for C-array (possibly nested).
Here, element type is `int`, storage is in-class 'intrinsic'
as a `int[4]` member so its single static bound is 4:

```c++
ray<int[4]> // Aggregate struct, data member int[4]
```

It is more idiomatically declared with a single `bounds` template argument:

```c++
ray<int,{4}> // Same type as ^^, data member int[4]
             // (braces may be elided; ray<int,4>)
```

The idiomatic way to declare a multi-dimensional array is then with its multiple extents collected in a single `bounds` argument.
The argument effectively groups a set of extents, their ordering, rank, etc., and specifies a multi-indexing scheme into a single contiguous range of elements.
As a concrete class type  a `bounds` argument can be implicitly list-initialized:

```c++
ray<E,{3,2}> // Aggregate struct, 2D-indexed as if 3x2
             // has 1D 'backing array' data member E[6]
```

The collected extents indicate continguous layout of elements.
Here, a pair of extents indicate it is accessed as if a 2D array; `3x2` in this case, 3 rows of 2 columns.
In other words, it contains a contiguous 1D 'backing' array of 6 elements but is accessed via a pair of indices with bounds 3 and 2.
The 1D offset is computed using the `bounds` values to combine the 2 indices, just as the compiler does automatically to index the equivalent built-in array (here, `E[3][2]`).

The elements are laid out in row-major order.
This is the same order as in the equivalent C-array - the difference is that layout is contiguous.
Contiguous layout of elements is usually what is wanted in a multi-dimensional array type.

<details><summary><b>C-array subobject semantics and lack of continuity</b></summary>

So-called multi-dimensional C-arrays are really nested 1D arrays.

The layout of `E[3][2]` is _not_ a contiguous range of `E`s; it is a contiguous 1D array of 3 contiguous 1D arrays of 2 `E`s.
The compiler is free to add padding between subarrays (in practice, `reinterpret_cast<E(&)[6]>` is contiguous as long as there is no padding).

Let's say that C-array has 'subobject' semantics.
The benefit of this layout is that any set of indices, of any rank, address an actual object; a subarray subobject.
However, this is only rarely what is wanted, the downside being that elements are now not strictly contiguous between the subarray objects.

The subobject semantics of multi-dimensional C-arrays is often suboptimal and it constrains usage.
In case this semantics is what is wanted it can be specified.
The following declarations are equivalent ways to specify a nested C-array member, `E[3][2]`;

```c++
ray<E[3][2]>  // Aggregate struct, data member E[3][2]
ray<E[2],{3}> // Exact same ^^^, data member E[3][2]
```

Note the reversed order of indices once they are separated (a more logical order in some ways).
The extents in a `bounds` collection are written in `{3,2}` order reflecting both nested C-array syntax and mathematical convention that a `3x2` array is 3 rows of 2 columns.

</details>

## Array value initialization, copy and assignment

Firstly, aggregate-init and copy-init work as expected for an array-holding aggregate:

```c++
// 1D array aggregate init & copy init
using int2 = ray<int,{2}>;        // Member int[2]
constexpr auto ci2 = int2{{0,1}}; // Aggregate init
auto i2 = ci2;                    // Mutable copy
```

Even if the `ray` represents multi-dimensional data,
here a `3x2` matrix, its 1D data member can only be initialised with a flat initializer list:

```c++
// 2D array aggregate init
using A = ray<int,{3,2}>;  // Member int[6]
auto a = A{{0,1,2,3,4,5}}; // Flat data init
```

There's no direct way to initialize from a nested init-list.
However, assignment operators accept an init-list rhs (the only operators that do),
and we'll see below how assignment can be used during initialization.

Aggregate initialization is discouraged
as it subverts the layout and indexing API by writing direct to storage.
It is unwieldy anyway for large matrices so is mainly useful in test code.
Whatever, it's useful, efficient, and can't be disabled (without breaking aggregate).
If you do use it then prefer not to elide braces on 1D arrays
as that's inconsistent with multi-dimensional arrays which do require correct nesting.

Note that braces _are_ elided from a copy-assign init-list:

```c++
// copy-assign
i2 = ci2;         // default copy-assign, lvalue rhs
i2 = int2{{1,0}}; // default copy-assign, rvalue rhs
i2 = {1,0};       // copy-assign overload, see below

//i2 = {{1,0}};   // Compile error, deleted overload
```

Copy-assign is part of the `ray` indexing API;
the right-hand side represents the index-view of the array class,
not the storage or layout.
In `i2 = {1,0};` the init-list initializes a notional `int[2]` array;
an overload `operator=(int const(&)[2])` is provided for this purpose.

Here, default copy-assign `operator=(int2 const&)` also matches `{1,0}`
_and_ non-brace-elided `{{1,0}}`.
The `int[2]` overload above is picked preferentially for a single-braced init-list
and a second deleted `int[1][2]` overload picks the double-braced case
to make it a compile error.



The `operator=()` overloads accept 'compatible'
C-array or `ray`,
i.e. with extents matching the `3x2` indexed view.
It even accepts init-list,:

```c++
// 2-phase init
A a;          // 1. default init indeterminate vals
a = {{0,1},   // 2. Assign nested init-list data:
     {2,3},   //    operator=(int const(&)[3][2])
     {4,5}};
```

The two-phase initialization can be inlined as a fused phased initialization:

```c++
// Inline constexpr correctly-nested two-phase init
constexpr auto c = A{}={{0,1},{2,3},{4,5}};
```

This 'fused-phased-init' is useful syntax for literals
as well as docs like this one and slide code.
It works by mutating an rvalue, here during constant evaluation.
The judge and jury may frown; it ain't illegal yet.

Equality `operator==` can be coerced with parens:

```c++
static_assert( c.t() == (A{}={{0,1},{2,3},{4,5}}) );
```

Otherwise, if preferred, call `operator==` explicitly:

```c++
static_assert( c.t().operator==({{0,1},{2,3},{4,5}}) );
```

We put it straight to work below.

## Transposed view

Member function `.t()` returns a transposed view of the array:

```c++
static_assert(
    c.t() == (ray<int,{2,3}>{} = {{0,2,4},
                                  {1,3,5}})
);
```

The view swaps the index order on access so that `c.t()`
appears as a `2x3` matrix.
No data is moved or mutated - `c` is constexpr immutable.
Indeed, there is no in-place transpose for a non-square static matrix
since that would have to change its static size.

What if you take a snapshot copy with the `.val()` member function?:

```c++
auto ct = c.t().val(); // ray<int,{{3,2},perm(1,0)}>
```

Then you still don't get transposed data,
you get an effective `memcpy` of the original `3x2` data
with the same swapped-index view on access.
It compares equal with the true `2x3` 'literal' above because they have the same index-view.
So they can also be assigned;

```c++
ray<int,{2,3}> ct = c.t();
```

Now we have transposed data, by strided copy into an array with compatible `2x3` shape.

## Column-major layout

This section introduces `perm(I...)`, the permutation 'instruction' or 'specifier',
to permute the order of extents;
`perm(1,0)` says to swap extents 0 and 1 while `perm(0,1)` is the default identity permutation.

The `bounds` argument,
so far only used to specify extents,
also encodes instructions like `perm` to change layout and indexing.
There are many equivalent ways to construct the same basic `3x2` array type:

```c++
ray<int,{3,2}> == ray<int,bounds{3,2}>
               == ray<int,extents{3,2}>
               == ray<int,{perm(0,1),{3,2}}>
         == ray<int,bounds{perm(0,1),extents{3,2}}>
```

The basic 2D layout & index permutations are:

```c++
using row_major  = ray<int,{extents{3,2}}>;
using col_major  = ray<int,{perm(1,0),extents{3,2}}>;
using transposed = ray<int,{extents{3,2},perm(1,0)}>;
```

Placed before the extents list, `perm` operates on the layout of data,
with indexing automatically adjusting to maintain the declared order of extents.
Placed after the extents, it operates only on the index order as seen by users.
Only one `perm` is permitted.

* Row-major is the default layout order for `ray`, as for C-array
* Column-major layout is achieved by permuting the layout order
* A transposed view is the result of permuting the index order

Now, `row_major` and `col_major` are both manifest `3x2` matrices so can be compared.
They compare equal when initialized with the same data via the indexing API:

```c++
static_assert( (row_major{} = {{0,1},{2,3},{4,5}})
            == (col_major{} = {{0,1},{2,3},{4,5}}) );
```

This is intended usage; the arrays should appear identical to all indexing operations.
We have to go behind the indexing API and aggregate initialize the storage to see the difference.

```c++
static_assert( row_major{{0,1,2,3,4,5}}
            != col_major{{0,1,2,3,4,5}} );

static_assert( row_major{{0,1,2,3,4,5}}
            == col_major{{0,2,4,1,3,5}} );

static_assert( row_major{ { 0, 1, 2, 3, 4, 5 }}
           == (row_major{}={{0,1},{2,3},{4,5}}) );

static_assert( col_major{ { 0, 1, 2, 3, 4, 5 }}
           != (col_major{}={{0,1},{2,3},{4,5}}) );
```

(Confused? You may want to review [Storage, layout and index view](#storage,-layout-and-index-view),
or [wikipedia](https://en.wikipedia.org/wiki/Row-_and_column-major_order),
or other reference. In short, column-major can be more efficient for column-wise access patterns
but otherwise appears the same from an indexing perspective.)

The transposed view appears as a `2x3` matrix so fails to compare with both row-major and column-major `3x2` matrices. They are incompatible for indexing purposes:

```c++
row_major{} == transpose{}; // Compile error, 3x2 vs 2x3
col_major{} == transpose{}; // Compile error, 3x2 vs 2x3
```



* `col_major` is laid out as `2x3` but indexed as the declared `3x2`
* `transpose` is laid out as `3x2` but indexed as `2x3`
  * declared `{3,2}` order is effectively swapped to advertise `{2,3}`;  
indexing-API consumers see a `2x3` matrix

### Assignment to permuted extents

The assignment operator= defined in the previous section works via the indexed view to assign a compatible-sized array or correctly-nested init-list.
For permuted layout or permuted views this implies strided access to the array elements:

```c++
// For permuted extents, indexed access is strided:
auto cm = col_major{} = {{00,01},{10,11},{20,21}};
/* copy assign! */ cm = {{00,10 , 20,01 , 11,21}};

auto tr = transpose{} = {{00,01,02},{10,11,12}};
/* copy assign! */ tr = {{00,10,01 , 11,02,12}};
```

The default copy assignment operator would still assign directly to the array member, unstrided.
The default and overloaded `=` assignments have different meaning and different cost!
So, the copy-assign special member is explicitly `=delete`'d when layout differs from indexed view.

Instead, we conscript `operator<<=()` to assign to the layout.
Access to the layout is always contiguous, i.e. non-strided, just as a deleted default assignment would be:

```c++
// Row-major has identical = & <<= non-strided access:
auto rl = row_major{} <<= {{00,01},{10,11},{20,21}};
auto ri = row_major{}  =  {{00,01},{10,11},{20,21}};
/* copy assign OK */ ri = {{00,01 , 10,11 , 20,21}};

// Layout is contiguous so its access is non-strided:
auto cm = col_major{} <<= {{00,01,02},{10,11,12}};
auto tr = transpose{} <<= {{00,01},{10,11},{20,21}};
```

This can be a confusing topic at first. Don't panic. Skip on and come back if needed.

Notice that there are many ways to declare the same array type with different levels of explicitness:

```C++
/*  ray<E,{3,2}> == ray<E,bounds{3,2}>
                 == ray<E,bounds{extents{3,2}}>
                 == ray<E,{perm(0,1),{3,2}}>
                 == ray<E,{{3,2},perm(0,1)}>
    == ray<E,bounds{extents{3,2},perm(0,1)}>  */
```

## More indexing

Recall that paren 'function call' syntax is used to index a `ray`:

* `a(i...)` multi-indexing, returns a reference-wrapper type `ray`

Let's punt on the exact return type and call it a 'ray-ref' for now.
The number of variadic indices must be less than or equal to the indexing rank.
Out-of-bounds access fails to compile in constant evalutation, otherwise it is undefined.

Given the same `3x2 E` array value `a` as before;

```c++
auto a = ray<int,{3,2}>{} = {{00,01},{10,11},{20,21}};
```

It can be indexed with zero, one or two indices:

```c++
auto r = a();     // ray-ref to full array 'a', 2D-indexed
auto a0 = a(0);   // ray-ref to 1st row {00,01}, 1D
auto a00 = a(0,0);// ray-ref to initial element (not int&)
```

Indexing a reference is no different to indexing a value:

```c++
auto r0 = r(0);   // Exact same as a0 ^^^ via ray-ref 'r'
auto r00 = a0(0); // ray-ref to initial element == a(0,0)
```

Here, the `auto` placeholder is a punt on the return type.
In practice, the actual type is rarely needed as indexing expressions are mostly used in assignments or other operations.

Note that `auto&` is _not_ correct for paren multi-index (it fails to compile).
`auto&&` will capture the prvalue return with lifetime-extension.

`a[i]` C-style indexing is defined only if the indexed subarray is a subobject in which case a language reference is returned
(consistent with C-array subobject indexing semantics):

```c++
//int& X = a[0];  // Rejected: no subobject at a[0]
int& l00 = a0[0]; // int& to initial element == a(0)[0]
```

Here, `auto` or `int` placeholder would copy the element value; to bind to the lvalue return requires `auto&`, `int&` or `auto&&` (which deduces `int&` and binds). `int&&` will fail to bind to an lvalue.

`decltype(auto)` is the best generic choice to take the return of an indexing expression

<details><summary>&nbsp<b><code>decltype(auto)</code></b></summary>

&nbsp;&nbsp;&nbsp;&nbsp;<b>`decltype(auto) a0 = a[0];`</b> is generically better.

For the C-array equivalent, `E a[3][2]{{00,01},{10,11},{20,21}};`  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto a0 = a[0]; `</b>` // (sub)array decays to pointer E*`  
C-array decay-to-pointer makes plain `auto` deduction a dangerous gotcha.

The returned C-subarray should be captured by reference to avoid decay:  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto&& a0 = a[0];`</b> ` // Reference to 1st row {00,01}`  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`decltype(auto) a0 = a[0];`</b> ` //  "" same`  
or, less generically, spell out mutability:  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto const& a0 = a[0];`</b> ` // "" const reference`  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto& a0 = a[0];`</b> ` // "" explicit mutable reference`  
or a compatible type:  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`E(&a0)[2] = a[0];`</b> ` // Specify exact type`  

Decay is a perrenial problem with C-array.  
After decay, array bounds are lost for the next level of iteration.

Lack of copy semantics is another problem with C-array.  
([P1997](https://wg21.link/p1997r1) _Relaxing Restrictions on Arrays_ proposes to allow C-array copy.  
&nbsp;P1997 calls for a compatible type or new `auto[]` deduction syntax:  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`E a0[2] = a[0]; `</b> ` // Copy of 1st row, specified type`  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto[] a0 = a[0];`</b> `// Copy of 1st row, deduced type`  )

`std::array` adds copy semantics; here's the equivalent;

&nbsp;&nbsp;&nbsp;&nbsp;`array<array<E,2>,3> a{{{00,01},{10,11},{20,21}}};`  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto a0 = a[0]; `</b>` // Copy of 1st row array{00,01}`  

**`auto`** says copy semantics; the rhs is array-like with copy semantics  
so we get a copy - now we need to capture by reference to disable copy;

&nbsp;&nbsp;&nbsp;&nbsp;<b>`auto&& a0 = a[0];`</b> ` // Ref to 1st row array{00,01}`  
&nbsp;&nbsp;&nbsp;&nbsp;<b>`decltype(auto) a0 = a[0];`</b> ` //  "" same`

In the `ray` case:  
&nbsp;&nbsp;**`auto`** is fine, but the reference semantics differs from `std::array`.  
&nbsp;&nbsp;**`auto&`** doesn't work as it cannot bind to the rvalue return.  
&nbsp;&nbsp;**`auto&&`** works generically in both cases, and indicates 'referenceness',  
but it forces reference deduction for a reference-wrapper rvalue  
(lifetime extension should make the reference-bind to rvalue safe).

&nbsp;&nbsp;&nbsp;&nbsp;<b>`decltype(auto) a0 = a[0];`</b> ` // Ref-like in all cases`
</details>

**`decltype(auto)`** is the generically correct, safe way to capture it.  
Plain **`auto`** does decay-copy for C-arrays. **`auto&&`** forces a reference on a ref-wrap value).

Here, non-fully indexed `ray`, `a[0]`, is a reference-wrapper type  
(so <b>`auto a0 = a[0]`</b> will work to `auto`-initialize `a0` in this case)  



`ray` also uses `operator()` for multi-indexing i.e. `a()`, `a(i)`, `a(i...)`.


A fully indexed array, `a(0,0)`, `a[0][0]`, `a(0)[0]`, `a[0](0)`, `a(0)(0)`,  
returns a reference to the indexed element (a language reference type -  
an lvalue reference for an lvalue array, rvalue for rvalue, const if const).


```c++
                  // auto& [a0,a1,a2] = a;
auto aT = a.t();  // Transpose indexing as if E[2][3]
auto aT0 = aT[0]; // Ref to 1st row of transpose {00,10,20}
                  // (views 1st col of 'a' with stride 2)
auto& [a00,a10,a20] = aT0;
```

No datum is mutated in the execution of this code.  
There is no actual transpose, no copy or swap of the data.  
Array indexing operations return reference types, as for built-in array;  
`a0`, `aT`, `aT0` are array-specialized reference-wrappers.  
They hold a pointer (or possibly a reference) to the underlying data.  
Here, they have static `bounds`, deduced to reflect their initializers

```c++
a = {{00,01},   // Assign new data to 'a' -
     {10,11},   // assignment accepts 'nested' braced-init
     {20,21}};  // correctly sized for the static bounds

aT0 = {3,2,1}; // Assign new data to 1st col of 'a'
```

Now data is mutated; array references 'assign through' to the data.  
Again, this is modelled on built-in array, when indexing an element

```c++
 // C-array


auto& a00 = a0[0];  // Reference to 1st element 00
a00 = 1;            // Assign new data to 1st element
```

### Design philosophy

`ray`, like `std::array` & `reference_wrapper`, is not a handle type  
(handle types hold a handle as well as handling ownership, growth, etc.).

Array value types wrap the storage itself, not a handle to storage.  
Array reference types are non-owning.

* Keep it low-level and simple for common cases
* Value semantics with references acting as lazy values
* Array value types are static-size aggregate; move is copy
* Array reference types may have dynamic extents
* Avoid copies by returning references when possible
* Avoid surpising reference aliasing

#### What it does and doesn't do

* Do layout, indexing & iteration with ranges support
* Do assignment and comparison
* Don't do container-like growth or handle-type ownership
* Don't do dynamic resizing (out of bounds) or Allocators

### Reference vs value

Array reference types are central.  
Array value types are secondary.

Reference-wrappers are small span-like types, suitable as arguments  
and cheap to construct as differently-indexed views of the storage.

Array values are handy to declare, initialize and return from functions  
(assuming copy-elison) and are most constexpr-friendly.

However, array values have rigid static bounds encoded in their type.  
That restricts mutation (e.g. matrix transpose may change its shape).  
And 

Maintaining value semantics for array reference types is useful.  
Assignment should assign through an array reference as if a value.  
Comparison should deep compare regardless of reference or value.

Built-in C-array doesn't assign or copy (unless wrapped or copy-captured)  
and functions cannot return C-array by value.

c..f proposal P1997 "Remove restrictions on array".

Comparison and equality is not defined for  (array args decay to pointer)

Indexing an array, whether value or reference, usually returns a reference  
but sometimes indexing has to perform a copy and return by value.

## Template parameter list

```c++
template <typename T, bounds... b> struct ray;
```

If the type `T` is a reference type (i.e. possibly cv-qualified `&` or `&&`)  
then `ray<T,b...>` is a reference(-wrapper) type.

```c++
int a[3][2];

auto lv{a}; // deduces ray<int(&)[3][2]>
            // with data member int(&)[3][2] = a

auto rv{decltype(a){}}; // deduces ray<int(&&)[3][2]>
                        // data member int(&&)[3][2]
                        // initialized to rvalue array

ray<int(*&)[3][2]> p{a}; // data member int(*)[2] = a
```

eg

```c++
template <typename E, bounds b0> struct ray<E,b0>;
```

The variadic `bounds` parameter is, in C++20, a class-type [NTTP](https://en.cppreference.com/w/cpp/language/template_parameters) that holds:

1. An extent or list of extents, e.g. `N`, `{}` or `{{},M,N...}`
2. A permutation of extents, for layout or indexing, and rank info

The empty brace `{}` represents a dynamic extent (for array reference types).

If `bounds...` is empty then the type parameter `T` specifies the array type.  
Otherwise, if `bounds...` is non-empty, then `T` specifies the element type.

(In C++17 and earlier, `bounds...` could be replaced by an `enum` type 'op code',  
specifying permutation and rank, and / or ranked `size_t...` extent specifiers.)

There's no equivalent of the AccessorPolicy or ContainerPolicy parameters  
from the `std`-proposed `mdspan` and `mdarray` types.

## 2D example

This example shows how to declare, initialize and index a 3x2 array.  
That's three rows of two columns. Let's fill with `int`s that hint at indices:

```c++
//              Logical 3x2 matrix
//       0 - based              1 - based

         { 00 01 }              { 11 12 }
         { 10 11 }              { 21 22 }
         { 20 21 }              { 31 32 }
```

First, declare types for C-style row-major or Fortran-style column-major storage.  
Both are logical 3x2 matrix types, indexed as if `int[3][2]`, by row then column

```c++
using row_maj32 = ray<int,{3,2}>;
using col_maj32 = ray<int,{perm(1,0),{3,2}}>; // transposed layout
```

```c++
row_maj C_style{ {0,1,2,3,4,5} }; // aggregate-init 1D C-array layout
col_maj Fortran{ {0,3,1,4,2,5} }; // aggregate-init 1D Fortran layout

C_style[1]; // {2,3}   reference type
Fortran(1); // {3,4,5} reference type
```

Both types are struct-wrapped `int[6]`; aggregates with 1D C-array storage  
so they can't be initialized directly as 2D arrays by nested braced-initializers  
(awkward in the column-major case as the initialization is always row-major).  
Two-phase initialization is a solution, e.g. default-init then `operator=`

```c++
auto C_style = row_maj{} = {{0,1},{2,3},{4,5}};
auto Fortran = col_maj{} =  {{0,1,2},{3,4,5}};
```

Here, `operator=(T const(&a)[M][N])` copies in argument `a` element-wise.  
(for column-major, source or destination memory access must be non-contiguous).  
(Note that two-phase initialization does _not_ preclude `constexpr` here.)

C-style `[I]` indexing returns a reference to the indexed subarray.  
If the indexed storage is a C-array then a C-array reference is returned

## Simple specializations

As a variadic argument, it can be empty:

```c++
   ray<A>
```

Here, `A` is an array specifier type; a C-array type or a ray-specifier type.  

Or, the variadic parameter can be provided a list of integer Args; `M,N...`

```c++
   ray<E,M,N...>
```

Here, `E` is the array element type (an object type)  
(this specialization should mangle the same in C++20, C++17 or earlier).

## Array reference types

We will name reference-wrapper types as `ray` specializations with  
a reference (or reference-like) type as the initial type argument:

```c++
ray<E(&)[N]>  // reference-wrapper 
ray<E*&,N>    // Static sized std::span<E,N> equivalent
ray<E*&,{}>   // Dynamic size std::span<E> equivalent
```

(the wrapped handle type will more often be a pointer than a reference)  
(or, read as `ray_ref<E,N>` or `ray_view` or `ray_span` or whatever).

Dynamic extents can only appear in array reference types.  
In other words, non-reference array types are always static-sized.

There is no owning 'handle' type; this is an exercise for the coder.



For over 60 years Fortran has demonstrated the utility of good 

In particular  class NTTP

(the more central non-owning array reference types come later).

array types

Variadic bounds... allows to specify C-array bounds directly in the type as e.g. ray<T[M][N]>

Inheritance may work here;template <typename T, atom A, bounds... b> struct atomic_ray : ray<T,b...> {};(haven't thought much yet on Accessor use cases).
Also note that there's no LayoutPolicy param - that's folded into the bounds NTTP...
boundsHere, bounds is a class type NTTP such that a ray bound can be initialized with:a single static extent; ray<T,N> (std::array<T,N> equivalent, holds T[N])a list of static extents; ray<T,{M,N}> (mdarray<T,N,M> equivalent, holds T[M*N])Case 2 is the multi-dim 'work horse'; a single list of static extents for contiguous layout.
Multiple bounds... produce a multi-dim C-array layout (hierarchical, generally discouraged):multiple single static extents; ray<T,N,M> (std::array<T[N],M> equivalent, holds T[M][N])
multiple general static extents; ray<T,{M,N},{m,n}> (holds T[m*n][M*N])
(an array type with empty variadic bounds ray<T[M][N]> also specifies multi-dim C array storage.)
My current bounds implementation is limited to hold a maximum of 8 extents - is that too limiting?(I recall hearing about some application that required 10 or 12 dimensions.)
Layout order / index orderbounds also holds a permutation that acts on the order of the indices(an arbitrary index order allows the column major case or any other).By default, bounds provides C layout with C index order (i.e. row major)(in fact a reverse-order permutation is needed for ray<T,{M,N}> to index like T[M][N]).
Note that there's no stride setup; I reckon that the combination of index permutation,along with taking different views on the data, will allow for any desired stride.
array reference types
Moving on to the non-owning reference / span types...
Let's use pointer-or-reference type specializations of ray itself; ray<T*,...>, ray<T&,...>These hold a pointer, or reference, of the specified type as well as any needed data.
Now, dynamic extents make sense in array_ref / span typesa single static extent; ray<T*,N> (static std::span<T,N> equivalent)
a single dynamic extent; ray<T*,{}> (dynamic std::span<T> equivalent)
a single list of general extents; (the mdspan equivalents)
ray<T*,{M,N}>, ray<T*,{{},N}>, ray<T*,{M,{}}>, ray<T*,{{},{}}>multiple bounds... e.g. ray<T*,{M,N},{{},{}}>A partial index into a multidimensional ray may return an array reference type:
   ray<int,{3,2}> a_ray{{0,1},{2,3},{4,5}};   a_ray[1];  // has type ray<int*,2> and refers to {2,3} subarray

Or, in the case of mutli-dim C-array storage, an actual C-subarray ref may be returned.
   int c_ray[3][2]{{0,1},{2,3},{4,5}};   ray<int(*)[3][2]> a_ref{c_ray};  // ref-wrap for existing C-array   a_ref[1];  // has type int(&)[2] and refers to {2,3} subarray



```C++
template <typename T, bounds... b> struct ray;

using E = /* Element: non-array object type; */
          /*   std::is_array_v<F> == false   */
          /*   std::is_object_v<F> == true   */;

ray<E>    // Not allowed; scalar F, with no bounds
ray<E[]>  // Not allowed; unknown-bound array type

// 1D ray specializations

ray<E[N]> // Wraps C-array (N==0 is non-standard, implementation-defined)
ray<E,N>  // equivalent, as std::array<F,N>, with N==0 specialization

// 2D ray specializations

ray<E[M][N]> // Wraps 2D C-array (discouraged; hierarchical layout)
ray<E[N],M>  // equivalent (note reversed order)
ray<E,N,M>   // equivalent (note reversed order)

ray<E,{M,N}> // Wraps 1D C-array F[M*N] indexed as 2D array F[M,N]


// 1D ray span specializations (could be a separate ray_ref type)

ray<E*&,N>    // Static sized std::span<F,N> equivalent
ray<E*&,{}>   // Dynamic size std::span<F> equivalent

ray<E(*)[N]> // Static sized, restrictive pointer, N==0 exception
ray<E(*)[]>  // Dynamic size, can only initialize from C-array [P0388]

             // Equivalent reference types (also && and const&)
ray<E(&)[N]> // Static sized, restrictive ref, N==0 excepted
ray<E(&)[]>  // Dynamic size, can only initialize from C-array [P0388]

ray<E&,N>    // Wraps static C-array reference 


// 2D ray span specializations ()

ray<E*,{M,N}> // 1D span of M*N Fs indexed as 2D array F[M,N]
ray<E*,N,{}>  // equivalent (note reversed order)



```



Array is an   storage for a [span](https://en.cppreference.com/w/cpp/container/span) of contiguous elements.
The difference is that an array is _embodied_, actually holding its elements as an intrinsic part.
A span is a mere portal constructed to serve its referenced range of elements.
Or, like vector, an array may hold its elements as an extrinsic part, via an owing handle.
Then what is the difference?

A universal array may hold value or it may _be_ a super span. The distinction is blurred. bringing indexing abilities

An array is mostly a [span](https://en.cppreference.com/w/cpp/container/span) of contiguous elements.
The difference is that an array is _embodied_, actually holding its elements as an intrinsic part.
A span is a mere portal constructed to serve its referenced range of elements.
Or, like vector, an array may hold its elements as an extrinsic part, via an owing handle.
Then what is the difference?

A universal array may hold value or it may _be_ a super span. The distinction is blurred. bringing indexing abilities
