# Sum Types

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | (your name and contact email address)                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

Adds various kinds of sum types to the language.
This enables nullability checks and discriminated unions.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

A lot of modern languages have language or very good library support for discriminated unions.
Also, a lot of modern languages distinguish nullable pointer and reference types from non-null ones.
D lacks both.

The proposal includes an in-depth analysis of what is expected of sum types
and argues that there is no one-size-fits-all solution,
therefore more than one construct is needed.

### Plus Types

The most basic case is a function that may return two kinds of otherwise unrelated values,
e.g. an `int` or a `string`.
The two possible result types have no particular order.

This leads to the type *n*-ary constructor `+`:
For *n* types <code>T<sub>0</sub></code>, …, <code>T*<sub>*n*−1</sub></code>,
<code>T<sub>0</sub> + … + T<sub>*n*−1</sub></code>
forms a *plus type.*
It’s called that because like the arithmetic addition,
the types commute and are associative.
The types have no order,
similar to how type constructors have no order
and therefore `const shared int` and `shared const int`  are the same type.
Also, like type constructors, having the same entity twice does not change the type:
`int + int` is `int` the same way `const(const int)` is `const int`.

A value is extracted from a plus type by referring to a component by type.

Plus types auto-inline with other plus types:
`(int + string) + Object` is the same as `int + (string + Object)`.
Therefore the analogy with associativity of the arithmetic addition.

The empty plus type is the bottom type `noreturn`
and adding the bottom type to a plus type doesn’t change it.

A plus type that includes `void` is `void`.

An easy way to form a plus type form the types of a compile-time sequence of types (aka. a type tuple),
simply add the type tuple `Ts` with `noreturn`.
In general, when a type tuple is connected with `+` to another type or type tuple,
all the types are connected to a plus type containing them.
Therefore, adding `noreturn` forms a plus type that,
unless the type tuple was empty,
does not contain `noreturn` as the bottom type is being removed.

Similar to how alias reassignment works,
a plus type can have a type added using `PlusType += Type`.
That is the same as `PlusType = PlusType + Type`
because of commutativity and associativity.

There is one type in particular that is going to be added to an existing type a lot: `typeof(null)`.
Because of that, we introduce the type suffix `?` so that `T?` means `T + typeof(null)`.
Therefore, `int?` is a nullable `int`.

Because plus types commute and associate,
`int? + string` is the same as `int + string?` and `(int + string)?`.
Also, `typeof(null)?` is `typeof(null)`.

A plus type `L` implicitly converts to another plus type `R` if
* either all types of `L` are also in `R`,
* or every types of `L` implicitly converts to a exactly one type in `R` best.

A type `L` that is not a plus type is treated as a 1-ary plus type
if it should be converted to a plus type.

A type `R` that is not a plus type is treated as a 1-ary plus type
if a plus type should be converted to `R`.

If a plus type `P` has reference-compatible components,
i.e. there is a type `T` such that the pointer type `T*` could refer to any of the components,
a `P` lvalue implicitly converts to an lvalue of type `T`.
E.g. the an lvlaue of type `int + immutable int` implicitly converts to an lvalue of type `const int`.

A plus type `L` is partially compatible with a plus type `R` if it does not implicitly convert,
but some constituent type of `L` is implicitly convertible to exactly one type in `R` best.

Casting plus types to another plus type that is only partially compatible requires an explicit cast.
The plus type object’s run-time type is checked if it is among the types that are implicitly convertible to exactly one type in cast-to type.
If it does not convert to any, it’s an error, and if it does not convert best to exactly one, it’s an ambiguity error.
However, not all conversions require checks to be done unsafely.
Only if the cast requires checks will it throw an `AssertError` if the run-time value is not convertible.

### Negative Types

A negative type is a type with the only purpose to interact with associative sum types.
It has no values, which means it cannot be realized in code execution, but it’s not the bottom type.

A negative type `-T` adds (cf. plus type) with `T` to the bottom type.
Its only purpose is to remove a type from a plus type.

The type `int + string + (-int)` is the same as `string`.

The most important application of negative types is `-typeof(null)`.
Adding it to nullable types removes the nullability.
Because that should be easy,
as for adding `typeof(null)` using `?`,
we add a type suffix for adding `-typeof(null)` to a plus type: `!`.
That is, `T!` is `T + (-typeof(null))`.
This is only useful if `T` already is a plus type that includes `typeof(null)`.
(E.g., the type `int!` cannot be realized
and the declaration of a variable of this type leads to a compile error.)
While possible, the purpose of `!` is not to be applied to non-nullable types.

To get non-nullable types backwards compatible into the language,
we add `!` types.

For class (or interface) types `C`,
pointer types `T*`,
function pointer types `R function(…) …`,
deletate types `R delegate(…) …`,
and associative array types `T[K]`
we re-interpret them as plus types:
* `C` is a shorthand for `C‼?` aka. `C‼ + typeof(null)`,
* `T*` is a shorthand for `T*‼?` aka. `T*‼ + typeof(null)`,
* `R function(…) …` is a shorthand for `R function‼(…) …?` aka. `R function‼(…) … + typeof(null)`,
* `R delegate(…) …` is a shorthand for `R delegate‼(…) …?` aka. `R delegate‼(…) … + typeof(null)`,
* `T[K]` is a shorthand for `T[K]‼?` aka. `T[K]‼ + typeof(null)`,

where
* `C‼` represents the newly added core-language type of non-null `C` references,
* `T*‼` represents the newly added core-language type of non-null pointers to `T`,
* `R function‼(…) …` represents the newly added core-language type of non-null function pointers,
* `R delegate‼(…) …` represents the newly added core-language type of non-null delegates,
* `T[K]‼` represents the newly added core-language type of non-null associative arrays with key type `K` and value type `T`.

The `‼` types are not proposed to be expressible directly in code,
which is why this document uses a non-ASCII character,
however they are easily expressible indirectly using the `!` suffix:
`T*!` expands to `T*‼ + typeof(null) + (-typeof(null))` which is `T*‼`.

Notably absent from this list are slices.
While slices can be `null`, they do not conceptually include `typeof(null)`;
rahter, an uninitialized slice is much more like an empty slice.
This also means that `T[]?` is different from `T[]` and has a very distinguished `null` value.

### Optional types

A plus type that includes `typeof(null)` is an *optional type*:
It has one distinguished value that represents a lack of data.

For this reason, optional types deserve special constructs that makes handling them especially convenient.

#### Memeber Access and Optional Member Access

The pseudo-syntax `{(…)}` indicates that parentheses may be present, but also may be omitted.

If `expr` is of plus type <code>T<sub>1</sub> + … + T<sub>*n*</sub></code> and
`expr.member{(…)}` does not compile as a UFCS call `member(expr, {…})`,
it compiles if <code>(cast(T<sub>*i*</sub>)expr).member{(…)}</code> compiles for each *i* = 1, …, *n*;
at run-time,
it executes the member access for the option that is present.
Any subset of the individual member accesses can in turn be UFCS calls.
The return type is <code>typeof((cast(T<sub>1</sub>)expr).member{(…)}) + … + typeof((cast(T<sub>*n*</sub>)expr).member{(…)})</code>.
Because plus types that include `void` are `void`,
if any of the member accesses returns `void`,
the whole expression returns `void`.
It is also `void` if there is no common type of those expressions.

If the type of `expr` includes `typeof(null)`,
in the non-UFCS case member access can never succeed,
as `typeof(null)` has no members.
For this reason, `expr?.member{(…)}` compiles if `(cast(typeof(expr)!)expr).member{(…)}` compiles;
at run-time, it performs a null-check and in the non-null case,
performs `(cast(typeof(expr)!)expr).member{(…)}`;
in the null case, evaluates to `null`.
The return type is therefore `typeof((cast(typeof(expr)!)expr).member{(…)})?`.
Because plus types that include `void` are `void`,
if any of the member accesses returns `void`,
the whole expression returns `void`,
and the null check does not change that
(in the null case, the result is technically `cast(void)null`).

#### Call and Optional Call

If `func` is a plus type <code>F<sub>1</sub> + … + F<sub>*n*</sub></code> composed of
function pointer, delegate and callable aggregate types,
the expression `func(…)` compiles if <code>(cast(F<sub>*i*</sub>)func)(…)</code> compiles;
at run-time,
it executes the call for the option that is present.
The return type is <code>typeof((cast(F<sub>1</sub>)func)(…)) + … + typeof((cast(F<sub>*n*</sub>)func)(…))</code>.
Because plus types that include `void` are `void`,
if any of the calls returns `void`,
the whole expression returns `void`.

If the type of `func` includes `typeof(null)`,
the call can never succeed,
as `typeof(null)` has no members.
For this reason, `func?(…)` compiles if `(cast(typeof(func)!)func)(…)` compiles;
at run-time, it performs a null-check and in the non-null case,
performs `(cast(typeof(expr)!)expr)(…)`;
in the null case, evaluates to `null`.
The return type is therefore `typeof((cast(typeof(expr)!)expr)(…))?`.

#### Indexing and Slicing, including Optional Ones

Basically the same as calls, except that the plus type be composed of
array, slice, and aggregate types with indexing/slicing operators.

Also, there will be `expr?[…]` which performs a null check and return `null` for `null`,
and lowers to an indexing expression of the active option otherwise.

#### Elvis Operator

The binary elvis operator `?:` connects objects of
plus type <code>L<sub>1</sub> + … + L<sub>*m*</sub> + typeof(null)</code>
and
plus type <code>R<sub>1</sub> + … + R<sub>*n*</sub></code>,
and returns an object of plus type
<code>L<sub>1</sub> + … + L<sub>*m*</sub> + R<sub>1</sub> + … + R<sub>*n*</sub></code>
with the semantics that:
If `lhs` is not `null` the elvis operator expression evaluates to `lhs` without evaluating `rhs`;
otherwise elvis operator expression evaluates to `rhs`.
Note that the `typeof(null)` is part of the type of `lhs`,
but not mentioned explicitly in the result type.
This means that `typeof(null)` is part of the result type only if it is present in the type of `rhs`.

It is right-associative, i.e. `a ?: b ?: c` means `a ?: (b ?: c)`.

Contrary to other languages, this elvis operator need not be given compatible types:
The type of `cast(int?)42 ?: "abc"` is `int + string`.
If the types are compatible, though, they collapse:
`cast(int?)null ?: 42` has type `int` because `int + int` is `int`.

#### Explicit Non-null Assertion

Add the postifix `!` operator.

The expression `expr!` asserts that `expr` is not `null` (throws `AssertError` if it is `null`),
otherwise evaluates to `expr`.
The type of `expr!` is `typeof(expr)!`.

#### Non-null Cast

Add `cast(!null)e` as a shorthand for `cast(typeof(e)!))e`.

If `typeof(e)` is a core-language reference type,
the cast does not require a check to be performed in the happy case,
and therefore no check is issued on platforms that trap `null` reads or writes,
even in `@safe` code.
On a platfrom such as WebAssembly that does not trap `null`,
a check is needed in `@safe` code.

If `typeof(e)` is not a core-language reference type,
a check is added in `@safe` code,
but not in `@system` code, e.g. an `int?` can be cast to `int`,
but in a `@system` function, the result may be garbage.
In particular, casting `bool?` with value `null` to `bool` in `@system` code
may result in values that appear to be `true` and `false`.
Inference of `@safe` considers this cast as `@safe`;
if the absence of the check is desired,
a programmer can wrap the cast with an immediately invoked `@trusted` lambda.

### Null-checking in Branching

For a object `x` of nullable plus type,
`if (x)` and `if (x !is null)` as well as `if (x is null)`,
has null-aware semantics:
In the branch in which `x` is not null,
an attempt is made to replace every usage of `x` by `cast(!null)x`.
If that succeeds, a hidden `auto ref` variable is introduced that is assigned `cast(!null)x`,
and a hidden function is introduced that returns the hidden variable.
The function returns by reference if and only if the hidden variable is a reference.
```d
if (x !is null)
{
    auto ref __not_null_x_ = cast(!null)x; // if possible, check is merged with condition
    static if (__traits(isRef, __not_null_x_)) ref __not_null_x() => __not_null_x_;
    else                                           __not_null_x() => __not_null_x_;
}
```
(The `auto ref` ensures that if `x` is of core-language reference type,
becasue the expression `cast(!null)x` is an lvalue,
it stays and lvalue.
In that case, `x` can even be re-assigned in the branch just not by a nullable value.
Otherwise, `__not_null_x()` is an rvalue and can not be re-assigned in any case.

If the rewrite fails, it is not performed.

In the branch where `x` is `null`,
if a compile-error occurs in the semantic phases,
an attempt is made to replace `x` by a `null` literal in the offending expression.

Even if a function is semantically entirely an `if` statement,
it may be written as:
```d
if (cond)
{
    ...
    return ...; // or
    throw ...; // or
    assert(0);
}
... effectively-else-branch ...
```
This is because control flow cannot exit the then-branch.
This can be made explicit at the cost of adding `else`, braces and indentation.

However, in declaration space, D already supports `else:`.
It means that everything that follows in the current scope
is to be placed in the `else` branch of the current branching construct
(`static if`, `version`, `debug`).

Add `else:` this to statement space.
It is a lot cheaper and mentally less burdensome than the aforementioned
`else` + braces + indentation.

However, it gives the compiler an explicit guarantee
that the rest of the scope is indeed part of the else-branch.

Without `else:`, the compiler may not assume a nullable value is not null,
even if it was checked.
```d
T? x;
if (x is null) return;
// In any case, `x` still of type `T?` as it was declared.
```
With `else:`, the case is clear:
```d
T? x;
if (x is null) return; else:
// If the code is friendly, `x` refers to the hidden function invocation
// and that returns a `T`, not a `T?`.
```

### Non-null references

In D, unbeknownst to many, references can be bound to `*null`.
A lot of programmers errouneously assume that references cannot be `*null`.

Contrary to other approaces towards backwards compatibility,
this should be reversed.
Almost all D code written assumes that `ref` parameters and return values aren’t `*null`.

Therefore, add `ref?` as a new storage class (together with `auto ref?`)
that explicitly allows binding `*null`.
From now on, a `ref` parameter must be bound to an lvalue with a non-null address.

### The Tristate Type

The type `bool?` is special in that it has only three values: `false`, `true`, and `null`.

We add tristate operators to handle these: `&&?` and `||?`.
For these operators, conceptually, `null` represents a value *between* `false` and `true`.
They convert any argument of a type other than `bool?` and `typeof(null)` to `bool`.

If you think of `false` as 0, `null` as ½ and `true` as 1,
`&&?` returns the minimum of the two arguments, and
`||?` returns the maximum of the two arguments,
where they do not evaluate the second argument if the first one already determines the result.

| *lhs*          | *rhs*           | *lhs* `&&?` *rhs*  |
|:--------------:|:---------------:|:------------------:|
| `false`        | *unevaluated*   | `false`            |
| `null`         | `false`         | `false`            |
| `null`         | `null`          | `null`             |
| `null`         | `true`          | `null`             |
| `true`         | *rhs*           | *rhs*              |

| *lhs*          | *rhs*           | *lhs* `||?` *rhs*  |
|:--------------:|:---------------:|:------------------:|
| `false`        | *rhs*           | *rhs*              |
| `null`         | `false`         | `null`             |
| `null`         | `null`          | `null`             |
| `null`         | `true`          | `true`             |
| `true`         | *unevaluated*   | `true`             |

Likewise, there are `&?` and `|?` added that have the same results,
but evaluate the second argument in every case.

The size of `bool?` is the same as `bool`.
The bit pattern representing `cast(bool?)null` is guaranteed to be
different from `0x00` and `0x01`,
but otherwise implementation defined.

This is so `cast(!null)nb` for `nb` a `bool?` can be an lvalue.

### Invalid Pattern Types

For built-in types `T`, there are four cases of how `T?` is created:
* If `T` is a core-language reference type, then `T?` is `T`
* If `T` is `X!` with `X` a core-language reference type, `T?` is `X`.
* If `T` is `bool`, `T?` is `bool?` with the special handling above.
* If `T` is some other fundamental type, `T?` is effectively a tuple of `T` and a `bool`,
where the `bool` is `true` if the `T` component is meaningful.

In every but the last case, the size of `T?` is the same as the size of `T`.

When a user-defiend `struct` type `T` falls conceptually into the category `int` and other fundamental types,
the addition of the `bool` value to make a `T?` is ideal.
However, some user-defiend `struct` types have invalid states.
Instead of wasting space, the whole object can be overwritten with an invalid pattern.

To indicate that an invalid pattern exists and ought to be used to model optional types,
one writes `enum null = ...;` into the struct.
Similar to `init` (which is automatically defined), `enum null` defines `enum typeof(this) __null`,
which for a plus type that includes exactly `T` and `typeof(null)`
guarantees that the size of `T?` is the same as `T`
and that `T.__null` will be used to mark the `null` value of the plus type.
Null checks are then `obj is T.__null`.

In the following example,
we define a struct that encapsulates a `size_t` to be used for a position.
A result of `size_t.max` is so unlikely that it can be used as a `null` value,
for example to be returned by some `Index? indexOf` function
that searches something and returns an index of where the searched-for object is,
but returns `null` when none is found.
The type system helps the user of the function to use and understand its result correctly.
```d
struct index_t
{
    size_t value;
    alias value this;
    enum null = Index(size_t.max);
}
```

In `@system` code, the cast to non-null is then a no-op.
The unchecked cast from such a possibly `null` type to `typeof(null)`
may be performed simply subtracting or `xor`-ing the `__null` bit pattern,
as the programmer takes responsibility that the object is indeed `null`
and therefore must have the bit pattern.

The null check effectively becomes a mere `is` comparison against `T.__null`.

### Discriminated Union Types

The next basic sum type is a discriminated union type.

The constituent types behave like elements of an array or tuple:
The order matters and duplicates are meaningful.

This leads to the type *n*-ary constructor `~`:
For *n* types *T*<sub>0</sub>, …, *T*<sub>*n*−1</sub>,
<code>(T<sub>0</sub> ~ … ~ T<sub>*n*−1</sub>)</code>
forms a *discriminated union type.*

Notably, `int ~ int` is not `int`,
and `int ~ string` is not `string ~ int`.
And in particular,
`int ~ string ~ Object` is neither `(int ~ string) ~ Object` nor `int ~ (string ~ Object)`.

A value is extracted by index.

Discriminated union types do not auto-inline:
`(int ~ int) ~ string` is different from `int ~ (int ~ string)`.

The empty discriminated union is the bottom type `noreturn`,
however, placing the bottom type in a discriminated union yields a different type.

A discriminated union that includes `void` is a compile-error.

Similar to how alias reassignment works,
a discriminated union type can be appended using `DU ~= Type`.
That ***not*** same as `DU = DU ~ Type`
because discriminated unions lack associativity:
```d
alias DU = int;
DU = DU ~ string; // DU = int ~ string
static if (bad)
    DU = DU ~ Object; // DU = (int ~ string) ~ Object
else
    DU ~= Object; // DU = int ~ string ~ Object
```

Using an explicit cast, a discriminated union converts to a plus type of the same types.
This operation is lossy if the discriminated union has duplicate types.

Using an explicit cast, a plus type converts to a discriminated union.

While removing types from a plus type can be done adding a negative type,
no such easy operation is available for discriminated unions
as in general, due to duplicates and order, it is not clear what exactly to remove.

However, due to ordering, discriminated union types can be indexed and sliced:
For a discriminated union type `DU`, <code>DU.Types[*i*]</code> is the index-*i* type (0-indexed),
and <code>DU.Types[*l*..*u*]</code> is a type tuple containing <code>DU.Types[*l*]</code>, …, <code>DU.Types[*u*-1]</code>.
When type tuples are concatenated using `~`, their types form the types for the discriminated union’s option.
An easy way to make a type tuple into a discriminated union type is to concatenate it using `~` with an empty type tuple.

That is, `DU.Types ~ AliasSeq!()` is the same as `DU`.

### Accessing Syntax

A plus type is accessed asking for the object of type `T`,
where `T` is among the constituent types of the plus type.

The most common way is case expression:
```d
(uint + string) x;
auto l = x switch
{
    case uint i   => i + 1;
    case string s => s.length;
};
// typeof(l) is size_t
```

Instead of an immediate expression, instead of `=>` use braces for a compound statement.
Inside a case’s compound statement block, use `switch return` to set the result of the switch expression
and skip the rest of the case’s block.
For nested `switch` expressions,
when you want to `switch return` not for the innermost `switch`,
put a label between `switch` and the opening brace in the respecitve `switch` expression,
and use the same label between `switch` and `return`:
```d
(uint + string) x, y;
auto l = x swtich outer
{
    case uint i => y switch
    {
        case uint j   => 0;
        case string t
        {
            switch outer return 0;
        }
    }
    case string s => 0;
};
```

If `typeof(null)` is a constituent of the plus type,
`case null` can be used for handling this case instead of `case typeof(null) unused`.

Of course, `default` is allowed to handle all remaining cases.

More intricate pattern matching is not needed and therefore not proposed.
It may be added to language by a later DIP.

A discriminated union, can be accessed the same if all its consituent types are different.
Otherwise it’s discriminated by index.
After `case` one uses an index enclosed in brackets to supply the index `i`,
and an optional variable name that binds to the option if it’s active.
The type of the variable is implicit and equal to `DU.Types[i]`.

```d
(int ~ int) x;
int y = x swtich
{
    case [0] i => i;
    case [1] j => j + 1;
};
```

### Enum Unions

This is both the backing construct the above lower to as well as the elaborate way to construct discriminated unions.

An `enum union` type is an enhanced `struct` type using the following `mixin template`:
```d
mixin template EnumUnion(string[] names, Ts...)
    if (1 <= names.length && names.length == Ts.length && Ts.length <= ubyte.max && 
        !{
            foreach (name; names)
                if (name.length >= 2)
                    return name[0 .. 2] == "__";
            return false;
        }()
    )
{
    static assert(is(typeof(this) == struct) || is(typeof(this) == class));

    import std.conv : __text = text;

    alias __Types = Ts;
    enum string[] __names = names;

    static if      (Ts.length <= ubyte .max) alias __Index = ubyte;
    else static if (Ts.length <= ushort.max) alias __Index = ushort;
    else static if (Ts.length <= uint  .max) alias __Index = uint;
    else                                     alias __Index = ulong;

    static foreach (i, name; names)
    {
        mixin(iq{enum __$(name)_t : __Index { index = $(i) }}.__text);
    }
    private __Index __index;
    invariant(__index < Ts.length);
    union
    {
        static foreach (i, name; names)
        {
            mixin(iq{ private Ts[i] __$(name); }.__text);
        }
    }
    static foreach (i, name; names)
    {
        mixin(iq{
            @trusted pure @nogc nothrow
            ref Ts[i] $(name)() @property
            {
                static immutable string errorMessage = i"EnumUnion: member `$(name)` (index $(i)) not active".__text;
                assert(__index == i, errorMessage);
                return __$(name);
            }
            $(is(typeof((Ts[i] x) @safe   { import core.lifetime : move; x = move(x); })) ? "@safe"   : "")
            $(is(typeof((Ts[i] x) @nogc   { import core.lifetime : move; x = move(x); })) ? "@nogc"   : "")
            $(is(typeof((Ts[i] x) nothrow { import core.lifetime : move; x = move(x); })) ? "nothrow" : "")
            $(is(typeof((Ts[i] x) pure    { import core.lifetime : move; x = move(x); })) ? "pure"    : "")
            void $(name)(Ts[i] value) @property
            {
                import core.lifetime : move;
                this.__index = i;
                auto self = (() @trusted => &this.__$(name))();
                *self = move(value);
            }
            $(is(typeof((ref Ts[i] x) @safe   { x = x; })) ? "@safe"   : "")
            $(is(typeof((ref Ts[i] x) @nogc   { x = x; })) ? "@nogc"   : "")
            $(is(typeof((ref Ts[i] x) nothrow { x = x; })) ? "nothrow" : "")
            $(is(typeof((ref Ts[i] x) pure    { x = x; })) ? "pure"    : "")
            void $(name)(return ref Ts[i] value) @property
            {
                this.__index = i;
                auto self = (() @trusted => &this.__$(name))();
                *self = value;
            }
        }.__text);
    }
    static foreach (i, name; names)
    {
        mixin(iq{
            $(is(typeof((Ts[i] x) @safe   { import core.lifetime : move; Ts[i] y = move(x); })) ? "@safe"   : "")
            $(is(typeof((Ts[i] x) @nogc   { import core.lifetime : move; Ts[i] y = move(x); })) ? "@nogc"   : "")
            $(is(typeof((Ts[i] x) nothrow { import core.lifetime : move; Ts[i] y = move(x); })) ? "nothrow" : "")
            $(is(typeof((Ts[i] x) pure    { import core.lifetime : move; Ts[i] y = move(x); })) ? "pure"    : "")
            this(__$(name)_t = __$(name)_t.index, Ts[i] $(name))
            {
                import core.lifetime : move;
                this.__index = $(i);
                this.__$(name) = move($(name));
            }
            $(is(typeof((Ts[i] x) @safe   { Ts[i] y = x; })) ? "@safe"   : "")
            $(is(typeof((Ts[i] x) @nogc   { Ts[i] y = x; })) ? "@nogc"   : "")
            $(is(typeof((Ts[i] x) nothrow { Ts[i] y = x; })) ? "nothrow" : "")
            $(is(typeof((Ts[i] x) pure    { Ts[i] y = x; })) ? "pure"    : "")
            this(__$(name)_t = __$(name)_t.index, ref Ts[i] $(name))
            {
                this.__index = $(i);
                this.__$(name) = $(name);
            }
        }.__text);
    }

    enum bool __anyDtor = {
        static foreach (alias T; Ts)
        {
            static if (__traits(hasMember, T, "__dtor"))
            {
                return true;
            }
        }
        return false;
    }();
    enum string __dtorAttribute(string attr) =
    {
        string result = attr;
        static foreach (i, alias T; Ts)
        {
            static if (__traits(hasMember, T, "__dtor"))
            {
                static if (!is(typeof(mixin(iq{(T x) $(attr) { x.__dtor; }}.__text))))
                    result = "";
            }
        }
        return result;
    }();
    static if (__anyDtor)
    {
        mixin(iq{
            $(__dtorAttribute!"@safe")
            $(__dtorAttribute!"pure")
            $(__dtorAttribute!"nothrow")
            $(__dtorAttribute!"@nogc")
            $(__dtorAttribute!"scope")
            ~this()
            {
                dtors: switch (__index)
                {
                    default:
                        break;
                    static foreach (i, alias T; Ts)
                    {
                        static if (__traits(hasMember, T, "__dtor"))
                        {
                            case i:
                                mixin("__",names[i]).__dtor();
                                break dtors;
                        }
                    }
                }
            }
        }.__text);
        alias __dtor_ = __dtor;
    }

    static foreach (alias dataMember; typeof(this).tupleof)
    {
        static assert(dataMember.stringof.length > 2 && dataMember.stringof[0 .. 2] == "__"
            || {
                foreach (name; names)
                    if (dataMember.stringof == name)
                        return true;
                return false;
            }(),
        "you cannot declare custom data members");
    }
    static assert(__traits(getOverloads, typeof(this), "__ctor").length == Ts.length * 2,
        "you cannot declare custom constructors"
    );
    static foreach (i, alias ctor; __traits(getOverloads, typeof(this), "__ctor"))
    {
        static assert({
            mixin(iq{alias Expected_$(i) = ref typeof(this) function(__$(names[i/2])_t = __$(names[i/2])_t.index, $(i % 2 != 0 ? "ref" : "") Ts[i/2] $(names[i/2]));}.__text);
            return is(typeof(&ctor) : mixin("Expected_", i.__text));
        }(),
        "you cannot declare custom constructors");
    }
    static assert(__anyDtor || !__traits(hasMember, typeof(this), "__dtor"),
        "you cannot declare a custom destructor"
    );
    static assert(!__anyDtor || __traits(isSame, typeof(this).__dtor, __dtor_),
        "you cannot declare a custom destructor"
    );
}

@safe pure nothrow @nogc
EU.__Index index(EU)(scope ref const EU eu) @property => eu.__index;
EU.__Index index(EU)(scope     const EU eu) @property => eu.__index;

template matchOrdered(fs...)
{
    auto ref matchOrdered(EU)(auto ref EU eu)
        if (fs.length == EU.__Types.length)
    {
        import std.traits, std.meta, std.conv;

        static if (__traits(isRef, eu)) alias valueOf = lvalueOf;
        else                            alias valueOf = rvalueOf;

        alias R = mixin({
            string result = "CommonType!(";
            static foreach (i, name; EU.__names)
            {
                result ~= iq{typeof(fs[$(i)](valueOf!(EU.__Types[$(i)]))),}.text;
            }
            return result ~ ")";
        }());
        final switch (eu.__index)
        {
            static foreach (i, name; EU.__names)
            {{
                static if (isFunction!(fs[i]) || is(typeof(fs[i]) == struct) || is(typeof(fs[i]) == union) || is(typeof(fs[i]) == class) || is(typeof(fs[i]) == interface))
                {
                    static assert(0,
                        "function pointer or delegate required: " ~
                        "use " ~ name ~ " => " ~ __traits(identifier, fs[i]) ~ "(" ~ name ~ ") " ~
                        "or " ~ name ~ " => " ~ name ~ "." ~ __traits(identifier, fs[i])
                    );
                }
                else static if (
                    (isFunctionPointer!(             fs[i]                ) || isDelegate!(             fs[i]                )) && is(typeof(             fs[i]                ) ps == __parameters)
                ||  (isFunctionPointer!(Instantiate!(fs[i], EU.__Types[i])) || isDelegate!(Instantiate!(fs[i], EU.__Types[i]))) && is(typeof(Instantiate!(fs[i], EU.__Types[i])) ps == __parameters))
                {
                    static if (!__traits(compiles,
                        mixin(iq{(ps[0 .. 1]){}($(name): valueOf!(EU.__Types[i]))}.text)
                    ))
                    {
                        static void fail(ps[0 .. 1])
                        {
                            static assert(0,
                                "name mismatch: enum union member name is `" ~ name ~"`, " ~
                                "but parameter name is `" ~ __traits(parameters)[0].stringof ~ "`"
                            );
                        }
                    }
                    else
                    {
                        case i:
                        static if (__traits(isRef, eu))
                        {
                            return cast(R) fs[i](
                                (function ref (ref EU eu) @trusted scope => mixin("eu.__", name))(eu)
                            );
                        }
                        else
                        {
                            import core.lifetime : move;
                            return cast(R) fs[i](move(
                                (function ref (ref EU eu) @trusted scope => mixin("eu.__", name))(eu)
                            ));
                        }
                    }
                }
                else
                {
                    static assert(0, "function pointer or delegate syntax required");
                }
            }}
        }
    }
}

template matchDefault(fs...)
    if (fs.length > 0)
{
    auto ref matchDefault(EU)(auto ref EU eu)
        if (fs.length - 1 <= EU.__Types.length)
    {
        import std.traits, std.meta, std.conv;

        static foreach (i, name; eu.__names)
        {
            static foreach (j, f; fs[0 .. $-1])
            {
                static if (isFunction!f || is(typeof(f) == struct) || is(typeof(f) == union) || is(typeof(f) == class) || is(typeof(f) == interface))
                {
                    static assert(0,
                        "function pointer or delegate required: " ~
                        "use " ~ name ~ " => " ~ __traits(identifier, f) ~ "(" ~ name ~ ") " ~
                        "or " ~ name ~ " => " ~ name ~ "." ~ __traits(identifier, f)
                    );
                }
                else static if (mixin(iq{
                    (isFunctionPointer! f                  || isDelegate! f                 ) && is(typeof(f                ) ps$(i)_$(j) == __parameters)
                ||  (isFunctionPointer!(f!(EU.__Types[i])) || isDelegate!(f!(EU.__Types[i]))) && is(typeof(f!(EU.__Types[i])) ps$(i)_$(j) == __parameters)
                }.text))
                {
                    static if (__traits(compiles,
                        mixin(iq{(ps$(i)_$(j)[0 .. 1]){}($(name): lvalueOf!(EU.__Types[i]))}.text)
                    ))
                    {
                        static if (is(typeof(mixin(iq{f$(i)}.text))))
                        {
                            static assert(0, i"double handler for option $(name)".text);
                        }
                        else
                        {
                            mixin(iq{alias f$(i) = f;}.text);
                        }
                    }
                    else
                    {
                        // pragma(msg, mixin(iq{(ps$(i)_$(j)[0 .. 1])}.text), i" is a handler, but not the handler for $(name)".text);
                    }
                }
            }
            static if (!is(typeof(mixin(iq{f$(i)}.text))))
            {
                mixin(iq{alias f$(i) = fs[$ - 1];}.text);
            }
        }
        enum string orderedFsStr = {
            string result = "AliasSeq!(";
            foreach (i; 0 .. EU.__Types.length)
            {
                result ~= i"f$(i), ".text;
            }
            return result ~ ")";
        }();
        //pragma(msg, orderedFsStr);
        alias orderedFs = mixin(orderedFsStr);
        import core.lifetime : forward;
        return matchOrdered!orderedFs(forward!eu);
    }
}

template match(fs...)
{
    auto ref match(EU)(auto ref EU eu)
        if (fs.length == EU.__Types.length)
    {
        static struct __X {}
        static noreturn f(T)(__X) { static assert(0); assert(0); };
        
        import core.lifetime : forward;
        return matchDefault!(fs, f)(forward!eu);
    }
}

template matchOrderedDefault(fs...)
    if (fs.length > 0)
{
    import core.lifetime : forward;
    import std.conv : text;
    alias matchers = fs[0 .. $ - 1];
    auto matchOrderedDefault(EU)(auto ref EU eu)
        if (fs.length <= EU.__Types.length + 1)
    {
        enum string defaultMatchers = () {
            string result;
            foreach (name; EU.__names[matchers.length .. EU.__Types.length])
            {
                result ~= iq{$(name) => fs[$ - 1]($(name)),}.text;
            }
            return result;
        }();
        mixin(iq{
            alias mo = matchOrdered!(matchers, $(defaultMatchers));
        }.text);
        return mo!EU(forward!eu);
    }
}
```

An `enum union` is split into data members and other members.
All data members must be simple declarations of the form `Type name`;
they cannot have default intializers.
Likewise, nested unnamed `struct`s or `union`s are not allowed.

Constructors are not allowed.

The declaration
```d
enum union EU
{
    // data members
    // other members
}
```
becomes
```d
struct EU
{
    mixin __enum_union!([names...], Ts...);
}
```

Let `Ts` be the types of the data members.
If `names` is an array containing the names of the data members 

## Prior Work
Required.

If the proposed feature exists or has been proposed in other languages, this is the place
to provide the details of those implementations and proposals. Ditto for prior DIPs.

If there is no prior work to be found, it must be explicitly noted here.

## Description
Required.

Detailed technical description of the new semantics. Language grammar changes
(per https://dlang.org/spec/grammar.html) needed to support the new syntax
(or change) must be listed. Examples demonstrating the new semantics will
strengthen the proposal and should be considered mandatory.

## Breaking Changes and Deprecations
This section is not required if no breaking changes or deprecations are anticipated.

Provide a detailed analysis on how the proposed changes may affect existing
user code and a step-by-step explanation of the deprecation process which is
supposed to handle breakage in a non-intrusive manner. Changes that may break
user code and have no well-defined deprecation process have a minimal chance of
being approved.

## Reference
Optional links to reference material such as existing discussions, research papers
or any other supplementary materials.

## Copyright & License
Copyright © 2024 by Quirin F. Schroll

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
The DIP Manager will supplement this section with links to forum discsusionss and a summary of the formal assessment.

```
EnumUnion is here for you

### Synopsis
```d
import enumunion;

struct EU
{
    mixin EnumUnion!(["x", "s", "y"], int, string, int);
    // your favorite members, except:
    //   - no constructors
    //   - no destructor
    //   - no data members
}

void main()
{
    auto eu = EU(x: 1);
    assert(eu.index == 0);
    assert(eu.x == 1);
    int n = 10;
    eu.match!(
        (long x) => writeln("x: ", x),
        s => f(s),
        y => n
    );
}
```

### FAQ

Q: Is it defensive?
A: Yes. If you use it incorrectly, problems are diagnosed.

Q: What members does it add?
A: Practically, it only adds `@property` accessors of the names you give it. Those check if the member is active.
Everything else it adds has two initial underscores, so there should be no name clashes with stuff you add to your type.

Q: Okay, but what does it add?
A: Here you go:
- The `alias __Types` and the `enum __names` to quickly access the construction.
- An alias `__Index`, the smallest unsigned type large enough to contain the index, given the number of members. In practice, it will be ubyte.
- A data member `__index` of type `__Index`.
- An unnamed union of the types and doubly underscored data member names you passed.
- The aforementioned `@property` accessors.
- An Enum type `__$(name)_t` for each option.
- Two constructors for each data member name (one with a `ref` parameter for copy and one without for move construction). It must be called using named argument notation using your data member name.
- A destructor if any of the members has a destructor.
- Some other `__` compile-time stuff to keep track of correct usage.

Q: Why the enum types?
A: To disambiguate constructors if two data members have the same type: `this(int x)` and `this(int y)` would not be different otherwise.

Q: Assuming data member `x` has a unique type, do I need to specify `x:` in the constructor?
A: Yes, intentionally so. The constructors take the uniquely typed and defaulted enum parameter as their first parameter. By using named arguments, you technically provide the second parameter first, and the remaining parameter has a default value, so you’re good to go. If you do not specify the parameter name, the argument would be bound to the first argument, which is of enum type, and overload resolution fails.

Q: How does `eu.index` work? Wouldn’t it have to be `eu1.__index`?
A: That’s because `index` is not a member function, but a UFCS call. That allows you to name an option `index` without internal stuff breaking. However, then `eu.index` accesses the option and you _must_ use `index(eu)` then to get the index, or `eu.__index`. Reading `__index` is safe.

Q: Can I assign through the `@property` accessors?
A: Yes, and it sets the internal index appropriately; i.e. to set an option, the option need not be active, but becomes active. Although the get accessors return by `ref`, normal-looking assignment uses the set accessor; if you need to, you can use `(ref () => eu.x)() = rhs` to assert that `x` is active and then set it directly.

Q: Is it `@safe`?
A: It’s not completely fool-proof. I don’t think it can be, honestly. The loophole is that the property accessors return my `ref`. They have to, otherwise two undesirable things happen: Accessing copies, which throws a wrench into non-copyable types and is potentially a performance hit. Worse, mutable objects cannot be mutated: You couldn’t do `++eu.x`, `eu.x.setFatal()` or something similar. This means you can store the address of an option in a pointer variable. While not inherently unsafe, if you change the active option, in general, dereferencing the pointer is now unsafe.

Q: Is `match` a module-level function called by UFCS as well?
A: Exactly.

Q: What if I don’t need to handle all options? Is there some kind of default case?
A: Use `matchDefault`; its last template argument is considered a default case that is applied to every option that is not covered by others, therefore its parameter’s name is irrelevat. It needs one parameter, though, even if you ignore it.

Q: How do I use them?
A: For `match`, you put in one handler of the form `option => expr(option)` for each option.

Q: Can a handler take the option by `ref` or `auto ref`?
A: Yes for `ref` and no for `auto ref`.

Q: How does it know what handler handles which option.
A: Names. You must provide handlers taking parameters with exaclty the same name as the options. To aid with this, the handlers must be function pointers or delegates and their parameter name must be equal to the option name. The only exception to this is the last handler on `matchDefault` which acts as a default handler; it still needs a parameter, which you’ll probably ignore most of the time.

Q: Do the handlers’ parameter types need to match exactly as well?
A: They don’t. A handler must be callable with the option, but if you write a handler for an option `x` that – for some reason – may be any signed integer type, you can just use `(long x) => expr(x)` for the handler; of course, `x` is then a `long` in any case.

Q: So, I can’t use functions or functors (object with `opCall`) for handling?
A: Not directly, but for a function `f`, just use `x => f(x)` or `x => x.f`; for a functor `f`, only `x => f(x)` works.

Q: But I can pass a function pointer or delegate directly?
A: Yes, but just because you _can,_ doesn’t mean you _should._

Q: Does it work with `@safe`, `pure`, `nothrow`, and `@nogc`?
A: Yes. As for DIP1000, it seems to work, but I haven’t figured out if there’s a bug regarding `scope` destructors. If your types don’t have destructors, you can definitely use it.

Q: Can I do something like algebraic data types with it?
A: No, that’s not intended.

Q: What are `matchOrdered` and `matchOrderedDefault` about?
A: They require that handlers be in the same order as the options. This reduces template bloat and compilation time, and it gives better diagnostics.

```