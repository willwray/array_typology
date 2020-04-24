# C++ Array Phenomenology and Typology

What is array?
The better question is "What is array for?".  
Whatever; what array is is what's on the menu today.  
What it is in C++20 and what it may be in C++23.

## Todays array

In C++, built-in array is an atypical yet phenomenal type.
Much maligned, resented as a vestige from C and with (deranged) calls to deprecate it in favour of std::array (or Ranges),
it is fundamental part of the core language.
The substrate underlying all objects is array of `char`.

On today's menu:

* For starters; [Phenomena](#array-phenomena)
  * A census of the senseless, restricted, irregular state of today's arrays
* Main course; [Core Typology](#core-typology)
  * A classification of built-in types associated with array
* Just desserts; [Class Typology](#class-typology)
  * A delicious assortment of array-like classes
* Post~~mortem~~prandial Epilogue;
  * [Val O'Ray](https://en.cppreference.com/w/cpp/numeric/valarray) sings Eulogy for the Irregular Span

On tomorrow's menu:

* A smörgåsbord; Universal Array

Frequent reference is made to proposal
[P1997](https://wg21.link/p1997) _Remove restrictions on array_

## Array Phenomena

* [Static size](#static-size)
* [Non-zero size](#non-zero-size)
* [Dynamic size](#dynamic-size)
  * [Array new](#array-new)
* [Multi-dimensional array, lack of](#multi-dimensional-array-lack-of)
* [Copy semantics, lack of](#copy-semantics-lack-of)
  * [Member array initialization conundrum](#member-array-initialization-conundrum)
* [Swap](#swap)
* [Assignment, lack of](#assignment-lack-of)
* [Comparison, lack of](#comparison-lack-of)
  * [Member array comparison via operator<=>()](#member-array-comparison-via-operator<=>())
  * [Wrappers for assignment and comparison](#wrappers-for-assignment-and-comparison)
* [Return from function, lack of](#return-from-function-lack-of)
  * [The not-so-great array return hack](#the-not-so-great-array-return-hack)
* [The Formal Parameter fiasco](#the-formal-parameter-fiasco)

### Static size

In C++ built-in arrays are static-sized (barring extensions, see [Dynamic size](#dynamic-size)).

### Non-zero size

Static size is non-zero (barring extensions).

The non-zero-size restriction could be relaxed for arrays now with C++20's `[[no_unique_address]]` attribute
(I don't know of a proposal),
useful in practice as it eliminates special cases.
E.g. `std::array`'s zero-size specialization is tricky to implement correctly with stable begin and end iterators.

Implementations already allow array of zero size as a [C extension](https://gcc.gnu.org/onlinedocs/gcc/Zero-Length.html) so there is existing practice.
Note that these are truly zero-sized,
`sizeof(int[0]) == 0`, unlike empty classes which must have sizeof at least one.

### Dynamic size

The C++ type system does not admit arrays of dynamic size (how could it; it's a static type system).

In C, local C-arrays may be of dynamic size;
[VLA](https://en.wikipedia.org/wiki/Variable-length_array),
[cppref](https://en.cppreference.com/w/c/language/array#Variable-length_arrays).
Most C++ compilers allow VLA, variable-length array, in C++ mode.

Attempts to add a dynamic array to C++ got confounded by the stricter object model, and ended in disarray...
The latest proposal [P0785](https://wg21.link/p0785) _Runtime-sized arrays and a C++ wrapper_ charts the sad story.

C also has [FAM](https://en.wikipedia.org/wiki/Flexible_array_member) 'flexible array member' in which a trailing array member of a struct may have unknown bound.
A C++ trailing FAM would be possible with [P0722](https://wg21.link/p0722) _Efficient sized delete for variable sized classes_
(in practice, using FAM as a non-standard extension is a straightforward workaround, at least for trivial element type).

#### Array new

So, in C++, dynamic size calls for dynamic allocation [CE](https://godbolt.org/z/Vz-bQY):

```c++
int n = 2;        // Dynamic size ... is:
//E e[n];         // Not accepted in standard C++
E* pe = new E[n]; // Accepted by operator new[]
delete(pe);       // (don't forget to delete).
```

Or placement `new[]` into a sufficiently-sized buffer (barely qualifies as dynamic) [CE](https://godbolt.org/z/CApZ6-):

```c++
char buf[8];
float* floats_ptr = new(buf) float[n];
std::destroy(floats_ptr,floats_ptr+n);
```

The `new`'d size may be zero, `new E[0]`, or deduced from the initializer as above.

Notes:
Compilers may remove or combine allocations as an optimization ('heap elision').
In C++20 `new` is allowed in constexpr context, though constexpr allocations can't persist to runtime.

### Multi-dimensional array, lack of

So-called multi-dimensional C-arrays are really nested 1D arrays.
The storage layout of `E[M][N]` is not a contiguous range of `E`s; it is a contiguous 1D array of `M` contiguous 1D arrays of `N` `E`s
(the compiler may add padding between subarrays).
C-array has 'subobject' semantics in that any set of indices, of any rank, address an actual object; a subarray subobject.
This is rarely needed, the downside being that non-contiguous layout
is constraining and often suboptimal for multi-dimensional usage.

One could imagine generalizing the built in array type with multiple static extents:

```c++
using int42 = int[4,2];
int42 i42{{1,2},{3,4},{5,6},{7,8}};
i42[1,2] = 12;
```

with `[4,2]` implying 8 elements in contiguous layout.
However, contiguous layout as a single `int[8]` object would mean that mid-rank subarrays are no longer objects;
what should `std::begin(i42)` return?


### Copy semantics, lack of

Built-in array copies only in very specific contexts [CE](https://godbolt.org/z/-krjEF) (any missing here?):

```c++
// string-literal copy-inits char array
char hi[6]{"hello"}, ho[]="ho";
struct C { char c[2]; } c{"c"};

// struct copy-init copies array members
struct A { int a[2]; } a{0,1};
auto b = a;

// auto structured binding copies array rhs
constexpr int a12[1][2]{{0,1}};
auto [a2] = a12; // mutable copy int[2]{0,1}

// lambda copy-capture copies arrays
auto& ha = [ho]()mutable->auto&{return ho;}();
```

P1997 proposes to allow array copy:

```c++
int a2[2]{0,1};
int b2[2] = a2; // copy-init from array rhs
b2 = {1,2};     // copy-assign from array rvalue
a2 = b2;        // copy-assign from array lvalue
```

### Member array initialization conundrum

Say you have an array member of a class.
How do you initialize it from a same-array-type constructor argument in the constructor initializer list?

None of the options are ideal [CE](https://godbolt.org/z/maTtAk):

```c++
// (1) Expand the array elements by hand
template <typename X> struct X4 {
  X x[4];
  X4(X const(&a)[4]) : x{a[0],a[1],a[2],a[3]} {}
};

// (2) Expand via extra variadic template constructor
template <typename X, size_t N> struct XN {
  X x[N];
  template <size_t... I>
  XN(X const(&a)[N], index_sequence<I...>)
         : x{a[I]...} {}
  XN(X const(&a)[N])
         : XN(a,make_index_sequence<N>{}) {}
};

// (3) For a single array member, reinterpret_cast
//     the arg to this class type and copy construct
template <typename X, size_t N> struct XxN {
  X x[N];
  XxN(X const(&a)[N])
   : XxN(*reinterpret_cast<XxN const*>(&a)) {} // !!
};

// (4) Give up and wrap the array member and the arg
template <typename X, std::size_t N> struct XW {
  struct wrapXN { X x[N]; } x;
  constexpr XW(wrapXN const& a) : x{a} {}
};

// (5) Give up and write a copy loop in the body
template <typename X, size_t N> struct Xcp {
  X x[N];
  Xcp(X const(&a)[N]) {for (auto& e:a) x[&e-a] = e;}
};

```

With P1997:

```c++
template <typename X, size_t N> struct XN {
  X x[N];
  XN(X const(&a)[N]) : x{a} {}
};
```

### Swap

There is a `std::swap` overload for array,
added in 2008 [LWG issue 809](https://cplusplus.github.io/LWG/issue809) _std::swap should be overloaded for array types_:
```c++
int a[2]{0,1}, b[2]{2,3};
std::swap(a,b); // In <utility> or <algorithm>
```

This works recursively for nested arrays [CE](https://godbolt.org/z/WQCEsR).

### Assignment, lack of

Built-in array does not assign:

```c++
int a[2]{}, b[2]{0,1};
a = b;         // error: invalid array assignment
```

Except as part of a class assign [CE](https://godbolt.org/z/FrfpzW):

```c++
class int2 {int a[2];}; // dummy class
(int2&)a = (int2&)b;  // cast to class and assign !!
```

P1997 permits `a = b;`

### Comparison, lack of

Built-in array does not compare!
Instead it decays to pointer and pointer comparison is done. Pointer comparison is deprecated. Warnings or errors can be raised [CE](https://godbolt.org/z/emXBet):

```c++
char hi[]{"hi"}, ho[]="ho";
if (hi != ho) // warning: comparison between two arrays
    throw;    //   is deprecated
if (ho=="ho") // warning: result of comparison against
    throw;    //   a string literal is unspecified
```

The deprecation allows for a future change in meaning.

Array comparison is not necessarily meaningful,
particularly for multi-dimensional data;
e.g. how should matrices of floating-point values compare?

#### Member array comparison via `operator<=>()`

Member `operator<=>()` for a class can be defaulted as an opt-in to compiler-generated comparisons [CE](https://godbolt.org/z/sjwJa4):

```c++
template <typename T, size_t N>
struct array {
  T data[N];
  constexpr auto operator<=>(array const& r) const = default;
};
static_assert( array<int,1>{0} == array<int,1>{0} );
```

The default comparison for array members is lexicographic comparison by element
(or deleted if the element type is not comparable).

#### Wrappers for assignment and comparison

Reference-wrappers that provide assignment and comparison operator overloads for arrays
can work around the lack of language support,
while keeping the operator syntax.

For assignment the class has to appear on the left-hand side.
For comparisons the wrapper class may be on either side (since C++20):

```c++
if (a < comparable_ref{b}) assignable_ref{a} = b;
```

An assignment wrapper that accepts same-type rvalue allows rvalue assignment:

```c++
  int a[2][2];
  assignable_ref{a} = {{1,2},{3,4}};
```

The implementation below recurses down nested C-arrays [CE](https://godbolt.org/z/wf_PQY):

```c++
template <typename T>
struct assignable_ref {
  using S = std::remove_cvref_t<T>;
  using U = std::remove_extent_t<S>;
  S& s;
  constexpr assignable_ref& operator=(S const& r)
  {
      if constexpr (std::is_array_v<S>)
          for (auto&& e : s)
              assignable_ref<U&>{e} = r[&e - s];
      else s = r;
      return *this;
  }
};
template <typename A>
assignable_ref(A&&) -> assignable_ref<A>;
```

### Return from function, lack of

Array is not permitted as a function return value.

The main reason is, or was, the lack of copy semantics.
However, 'guaranteed copy elision' in C++17 mandates no-copy RVO, return value optimization, for rvalue returns.
Eager decay-to-pointer of an rvalue array is avoided by binding to a reference, `&&` or `const&`:

```c++
using A = int[3]; // Type to create an rvalue array:
A&& a = A{1,2,3}; // can init rvalue-reference
//A a = A{1,2,3}; // can't init array from rvalue
```

The reference-binding extends the lifetime of the return value.  
(Note that `auto&&` placeholder works here while `decltype(auto)` fails for now.)

P1997 permits array return from function.

```c++
auto iota4() -> int[4] { return {1,2,3,4}; };
decltype(auto) i4 = iota4();
```

(`decltype(auto)` works here assuming P1997 copy-init.  
 `auto[]` placeholder is also proposed for array-constrained deduction.)

#### The not-so-great array return hack

This hack simulates array return from function by returning a reference to defaulted argument,
itself a lifetime-extended rvalue reference to array.
To avoid the compiler detecting any dangle as UB,
and compiling it away,
the returned array is immediately copied by structured binding
(which calls for another level of array-wrap unless you want to expand and bind the elements themselves) [CE](https://godbolt.org/z/3wHSaf):

```c++
template <size_t N>
constexpr auto iota(int(&&a)[1][N]={}) -> int(&)[1][N]
{
    for (auto& e : a[0])
        e = &e - a[0];
    return a;
}
for (auto [iota4] = iota<4>(); auto d : iota4)
    putchar('0' + d);  // 0123
```

P1997 would allow to drop all these UB-riddled tricks:

```c++
template <size_t N>
constexpr auto iota() -> int[N]
{
    int a[N];
    for (auto& e : a)
        e = &e - a;
    return a;
}
for (auto d : iota<4>())
    putchar('0' + d);  // 0123
```

### The Formal Parameter fiasco

The less said about this the better.

```c++
int len(char a[]) { return sizeof a - 1; }

int main() { return len("bye-bye"); }
```

This program computes strlen of "bye-bye", correctly on most platforms...

As with function parameters, template 'NTTP' parameters of array type are also silently adjusted-to-pointer,
ingoring any extent.
(Can't blame that on C.)

Maybe, one day, this will be the former Formal Parameter fiasco;
there are moves underway towards deprecation.

## Core Typology

Let `E` be the array element type, any cv qualifers included,  
and `N` the array extent, usually taken to be of type `size_t`.

* [Core array `E[N]` and its associated compounds](#core-array-E[N]-and-its-associated-compounds)
  * [`E[]` : Unknown bound / unbounded array type](#unbounded-array)
  * [`E[N]` : Bounded array](#bounded-array)
  * [`E(&)[]` &nbsp;&nbsp;: Reference to unbounded array<br>`E(&)[N]` : Reference to array](#reference-to-array)
  * [`E(*)[]` &nbsp;&nbsp;: Pointer to unbounded array<br>`E(*)[N]` : Pointer to array](#pointer-to-array)

### Core array `E[N]` and its associated compounds

`E[N]` is a crystal formed of `N` contiguous constituent elements.  
Its most infamous associated compound is decay product `E*`.  
The related types are pointer `*` and reference `&` to array, both  
bounded and unbounded, and the unbounded array type itself;  
`E(*)[N]`, `E(*)[]`, `E(&)[N]`, `E(&)[]`, `E[]`

#### Associated compounds  

```c++
            E(*)[N] -- E(*)[]       No decay
          /
array E[N] - - - - - - E[]  Unbounded
          \
Decay       E(&)[N] -- E(&)[] -> E*  pointer
```

Weaker types to the right, losing extent and then, reaching  
bottom, losing array-ness with full decay-to-pointer type `E*`

<h4 id="unbounded-array"><code>E[]</code> : Unknown bound / unbounded array type</h4>

Unbounded array type `E[]` is only usable as a type  
i.e. it is not an object type; there can be no object of the type.  
It is an incomplete type in that a declaration with this type is  
incomplete and a definition must be completed with a bound:

```c++
template <typename T>
concept complete = requires { sizeof(T); };

extern int ub[];
static_assert( not complete<decltype(ub)> );

int ub[40];
static_assert( complete<decltype(ub)> );
```

On the other hand, references `E(&)[]` or pointers `E(*)[]`  
to unbounded array _are_ complete types themselves; they refer  
to the initializing array whose known extent is lost in binding  
(examples below).

As a convenience, unbounded array syntax may be employed  
in a definition to allow deduction of size from the initializer:

```c++
int x[]; // error: needs size or initializer

int x[]{1,2,3}; // Size deduced from initializer
//   ^^ Not an unbounded array; decltype(x)=int[3]
```

The apparently incomplete type is completed immediately  
 by the initializer so the actual declared type is complete.

<h4 id="bounded-array"><code>E[N]</code> : Bounded array</h4>

The array value type `E[N]` is an object type, a simple compound  
aggregate type, a contiguous sequence
of one element type.

Aggregate initialization, from a braced initializer list, is the only  
way to initialize a value of array type (currently, see [exceptions](#copy-semantics-lack-of)).

Values of array type are unstable in use, eagerly decaying to a  
pointer to the first element of the array.

For example, attempts to pass an array by-value to function  
induce decay-copy of the array argument, i.e. decay-to-pointer  
(see [The Formal Parameter fiasco](#the-formal-parameter-fiasco) for this nasty C++ gotcha)  
(if it were possible to pass array by value then it would incur a  
costly copy for large arrays so could be a performace gotcha).  

<h4 id="reference-to-array"><code>E(&)[]</code> &nbsp;&nbsp;: Reference to unbounded array<br>
<code>E(&)[N]</code> : Reference to array</h4>

References act as aliases to their bound array;  
they act just as the array itself, including decay-to-pointer.

Pass-by-reference is the classic idiomatic C++ way to pass arrays  
(C doesn't have reference types so it's pass-by-pointer in C)  
(C++ now has `std::span` and Ranges as alternatives).

Const lvalue or rvalue ref `const&`, `&&` can bind to rvalue arrays.  
This includes initializer lists, which are materialized to temporary  
array rvalue, which then bind with lifetime extension:

```c++
int (&&rv3)[3] = {1,2,3}; // int(&&)[3]
```

C++20 allows reference-to-unbounded-array `E(&)[]` to bind to  
array values ([P0388](https://wg21.link/p0388) _Permit conversions to arrays of unknown bound_):

```c++
int (&&rvu)[] = {1,2,3};     // int(&&)[]
int (&lvu)[] = rv3;          // int(&)[]
char const(&stru)[] = "C++"; // char const(&)[]
```

Unbounded array is _not_ being employed here for size-deduction;  
reference-to-unbounded _is_ the declared type of the variable.  
The extent is lost in binding, 'size-erased' away from the initializer.

<h4 id="pointer-to-array"><code>E(*)[]</code> &nbsp;&nbsp;: Pointer to unbounded array<br>
<code>E(*)[N]</code> : Pointer to array</h4>

Pointers are more regular than references;

* Pointer assignment simply reassigns the pointer value  
(a reference assigns-through to referee)  
(a reference itself cannot be 'reseated' post initialization)
* Pointers can be stored in arrays or other containers  
* Pointer members don't disable class-default copy-assign  

However, unlike a reference, which acts as an alias to referee,  
pointer `p` is one compound level above pointee `a`;

* address-of is needed to initialize `auto* p = &a;`
* then a dereference prior to use `*p`

Other differences:  
Pointer-to-array is immune from array decay-to-pointer.  
Pointers are nullable so can represent optional or exceptional value.

Apart from the preceding differences,
pass-by-pointer-to-array  
is more or less equivalent to pass-by-reference-to-array.

### Declared types and initializations

Array type declarations and definitions have 'noisy' syntax.  
Using `auto` where possible helps a little:

```c++
E a[N]{0,1};  // a : E[N]    Array 

auto* p = &a; // p : E(*)[N] Pointer to bounded array
E(*q)[] = &a; // q : E(*)[]  Pointer to unbounded array
auto& r = a;  // r : E(&)[N] Reference to bounded array
E(&u)[] = a;  // u : E(&)[]  Reference to unbounded array
```

Using `using` helps more. Here are the same declarations:

```c++
using A = E[N];
using U = E[];

extern A a;   // a : E[N]    Array

extern A* p;  // p : E(*)[N] Pointer to bounded array
extern U* q;  // q : E(*)[]  Pointer to unbounded array
extern A& r;  // r : E(&)[N] Reference to bounded array
extern U& u;  // u : E(&)[]  Reference to unbounded array
```

The variables declared here are used as an ongoing example.  
Here are definitions with initializations (code is here [CE](https://godbolt.org/z/_XzR_L)):

```c++
A a{0,1};  // a : E[N]    Array value, aggregate init

A* p = &a; // p : E(*)[N] Pointer to bounded array
U* q = &a; // q : E(*)[]  Pointer to unbounded array
A& r = a;  // r : E(&)[N] Reference to bounded array
U& u = a;  // u : E(&)[]  Reference to unbounded array
```

Pointers `p`, `q` or references `r`, `u` to array may only be initialized  
by an array, not by an `E*` pointer, so they are 'stronger' types.

Initializations of `q` and `u` are newly allowed in C++20  
[P0388](https://wg21.link/p0388) _Permit conversions to arrays of unknown bound_  
(P0388 is only implemented in gcc at the time of writing).

### Decayed types

An `auto` declaration does 'decay-copy' from the initializing type:

```c++
auto d = a;  // E* <- E[N] array decay-to-pointer E*

auto dp = p; // E(*)[N] No decay for pointer-to-array
auto dq = q; // E(*)[]  No decay for pointer-to-array
auto dr = r; // E*      Decay for reference-to-array
auto du = u; // E*      Decay for reference-to-array
```

Array value `a` suffers decay, as do references to array, `r`, `u`.  
Pointers-to-array avoid decay by already being pointers, `p`, `q`.

### Usage: indexing

References to array `r`, `u` and decayed pointer value `d`  
all behave exactly as array `a` itself in indexing usage.

```c++
int& ai = a[i];
int& di = d[i];

int& pi = (*p)[i]; // pointer-to-array, must deref
int& qi = (*q)[i]; // pointer-to-array, must deref
int& ri = r[i];
int& ui = u[i];
```

Pointers to array, however, are one compound-level above array;  
initialization had to take the address of the target array, `p = &a`,  
and, now, before index usage it must be dereferenced as `(*p)[i]`  
(parens needed as index `[i]` binds stonger than dereference `*`).

### Pointer ambiguity

By including pointer-to-bounded-array `E(*)[N]` as an associated type  
above we also introduce an ambiguity with the decayed pointer `E*`;  
given one of these pointer types then what is the element type?  
There's no ambiguity if only 1D arrays are in play.  
Otherwise, more information is required in order to disambiguate.

### Properties



* `E[N]` immediate, static, owning, no copy* or assign, ~~ptr compare~~, decay
* `E(&)[N]` handle, static, non-owning, no copy or assign, ~~ptr compare~~, decay
* `E(&)[]` handle, static, non-owning, no copy or assign, ~~ptr compare~~, decay
* `E(*)[N]` handle, static, non-owning, shallow copy & assign, ptr compare
* `E(*)[]` handle, static, non-owning, shallow copy & assign, ptr compare
* `E*` handle, dynamic, non-owning, shallow copy & assign, ptr compare
* `E*` handle, dynamic, owning, shallow copy & assign, ptr compare  
  * `= new E[N]` 'owning' in that it points to an allocation


Array-related types, listed from strongest, most static, to weakest, most dynamic:

|Type     |Ref   |S/D   |Own|copy/`=`   |`==`      |Decay   |
|----     |:----:|:----:|:-:|:------:   |---       |:--:    |
|`E[N]`   |value |static|own|no* no     |ptr*      | :bomb: |
|`E(&)[N]`|handle|static|no |no* no     |ptr*      | :bomb: |
|`E(&)[]` |handle|static|no |  no       |ptr*      | :bomb: |
|`E(*)[N]`|handle|static|no |ptr :bomb: |ptr :bomb:|        |
|`E(*)[]` |handle|static|no |ptr :bomb: |ptr :bomb:|        |
|`E*`     |handle|dyn   |no |ptr :bomb: |ptr :bomb:|        |
|`E* =`<br>`new E[n]`     |handle|dyn   |own|ptr :bomb: |ptr :bomb:|        |

