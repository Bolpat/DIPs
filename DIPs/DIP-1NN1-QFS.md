# Compile-time Indexing Operators

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | *TBD by DIP Manager*                                            |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Quirin F. Schroll (q.schroll@gmail.com)                         |
| Implementation: | *none*                                                          |
| Status:         | Draft                                                           |

## Abstract

In the current state of the D Programming Language, a user-defined type can only
define indexing operators with function parameters.
This DIP proposes additional compiler-recognized member names to implement indexing
where the indexing parameters are required to be – or handled differently when – known at compile-time.
Such a solution has been suggested independently by Eyal on the issue tracker.[⁽¹¹⁾]

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Alternatives](#alternatives)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [References](#references)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

When implementing heterogeneous structures like tuples, indexing syntax can only be implemented in a very narrow way
(cf. [current state alternatives](#alternatives)).
Given a user-defined tuple type, sophisticated indexing would require handling the indexing parameters as
compile-time values to determine even the type of the indexing expression.

With the additions proposed by this DIP, it would be possible to make
e. g. slices of Phobos’ `Tuple` return a `Tuple` again.

With what this DIP proposes, custom types can provide the full range of syntactic possibilities of dynamic indexing
also for the cases when the *indexing parameters* (= the expressions in square brackets) should be determined at compile-time
and made available to the handling members as template parameters instead of function parameters.

As an example, the expression `object[i]` would be rewritten as `object.opStaticIndex!i` first.
If that succeeds, the handling member `opStaticIndex` can use the value of `i` as a compile-time value.
If that fails, `object[i]` will be rewritten as `object.opIndex(i)`, where `i` is a function parameter.

As an additional benefit, even if not required, the custom type can check compile-time known indexing parameters
for validity by overloading both *dynamic* and *static* indexing operators
(for the definition of these terms see [Description](#description)).
This is what the compiler effectively does for static arrays:
While, formally by the specification, indexing strictly is a run-time operation, when using a compile-time known index that is out of bounds,
the compiler will report an error nonetheless.
With what this DIP proposes, it is possible to implement this bug-preventing feature in user-defined types.

With this DIP, the language converges more to the state where language- and programmer-possibilities converge.
The author believes that one of the long-term goals of the D Programming Language should be
that user-defined and language-given constructs should not be distinguishable from a user perspective
and that the language supplies the necessary and convenient tools
for user-defined constructs to mimic almost-any behavior some the language-specified constructs exhibit.

It is worth stressing that this DIP introduces user-defined syntax that changes semantics based on compile-time
availableness of syntactically identical arguments.
It can be used to implement functions that are aware of compile-time availableness to some parameters.
This functionality is no addition the language; template alias parameters can bind to names that hold run-time values.
Subsequent checks for their compile-time availableness are possible via `__traits(compiles, ...)`.

An example of awareness of compile-time availableness is the concept of infinite ranges.
It relies on having the member `empty` as a compile-time constant that is equal to `false`.
As a consequence some functions act differently on a range that is known at compile-time to be infinite,
compared to just acting on a range that happens to be infinite at run-time.

## Prior Work

There is no prior work known to the autor of this DIP.

For the subsection [Compile-time Function Parameters](#compile-time-function-parameters) of the [Alternatives](#alternatives) section,
it should be mentioned that Walter Bright had a slide in a talk proposing a `static` function parameter storage class
that, as far as the author understood it, should only bind to compile-time available values.
That way, functions would overload on the compile-time avalialbleness of values.

## Description

### Current State

When a user-defined type has any of the following compiler-recognized members
* `opIndex`,
* `opIndexUnary`,
* `opIndexAssign`,
* `opIndexOpAssign`,
* `opSlice`, and
* `opDollar`,

the compiler generates appropriate rewrites using the aforementioned members
to provide indexing for that type when using indexing syntax.[⁽¹⁾]

### Term Definitions

These member functions, except `opDollar`, will be called *dynamic indexing operators* in the following,
in contrast to the *static indexing operators* proposed by this DIP.

The DIP proposes to add the following names to the compiler-recognized member names for operator overloading:
* `opStaticIndex`,
* `opStaticIndexUnary`,
* `opStaticIndexAssign`,
* `opStaticIndexOpAssign`, and
* `opStaticSlice`.

Notably absent is the name `opStaticDollar`.
By the proposed changes of this DIP, the rewrite of `$` remains unchanged,
as `opDollar` does not require any function parameters.[⁽¹⁾]
There are multiple examples in Phobos defining `opDollar` not by a function,
but e. g. an `enum`.

Other used terms already given in the D Language Specification here include
* the *overloadable unary index operators* are `+`, `-`, `~`, `*`, `++`, `--`,
* the *overloadable binary index operators* are `+`, `-`, `*`, `/`, `%`, `^^`, `&`, `|`, `^`, `<<`, `>>`, `>>>`, `~`.

Furthermode, an indexing expression is called *one-dimensional* if it is of the form
`expression[index]` where `index` is a single expression (i. e. no commas involved)
that is not lowered to an empty sequence or a sequence with 2 or more elements.

### Main Proposal: Rewrites of Indexing Expressions

The DIP proposes that when rewriting indexing expressions
the compiler will first try static indexing operators,
and if those rewrites fail, try rewrites according to dynamic indexing operators
as currently described in the D Language Specification.[⁽¹⁾]
The DIP does not propose to change the rewrites to dynamic indexing operators.

The rewrites to static indexing operators is completely analogous to the dynamic ones,
with the main difference that the arguments inside the square brackets will be
plugged in as template arguments instead of function arguments.
The other arguments, notably only the right-hand sides of index assignment operators,
are unaffected and will be run-time function arguments.

* The expression `op object[indices]`,
where `op` is an overloadable unary index operator,
is rewritten as `object.opStaticIndexUnary!(op, indices)()`
or as `object.opStaticIndexUnary!(op, indices)`
if the call in the former rewrite fails.
* The expression `object[indices] op= rhs`,
where `op` is an overloadable binary index operator,
is rewritten as `object.opStaticIndexOpAssign!(op, indices)(rhs)`.
* The expression `object[indices] = rhs`
is rewritten as `object.opStaticIndexAssign!(indices)(rhs)`.
When `indices` is the empty sequence and this rewrite fails, the expression
is rewritten as `object.opStaticIndexAssign(rhs)`,
i. e. without template instantiation syntax.
* The expression `object[indices]`, if not preceded by a unary operator or followed by some assignment operator, or
if so, when the above rewrites failed, will be rewritten as `object.opStaticIndex!(indices)()`
or as `object.opStaticIndex!(indices)`
if the call in the former rewrite fails.
When `indices` is the empty sequence and this rewrite fails, the expression
is rewritten as `object.opStaticIndex()`,
or as `object.opStaticIndex`,
if the call in the former rewrite fails,
i. e. without template instantiation syntax.

If all static rewrites fail, rewrites to dynamic indexing operators are tried.

Note that in cases where the rewrites do not require template instantiation syntax,
the implementation could be done using dynamic indexing operators.
Those rewrites are mainly included for consistency.

<!---
It could be argued to first try unary dynamic indexing rewrites and binary assignment dynmaic indexing rewrites
before static indexing rewrites without operators.
The author sees this as a minor corner case, as overloading unary dynamic indexing and not unary static indexing,
but plain static indexing, is an odd thing to do.
Since this is possible, some decision must be taken, and the proposed one has the advantage
that in the explanation of rewriting process of indexing expressions can be prepended by
*In any case, static indexing operators are preferred*.
<!--- Possibly needs expansion --->

Indexing parameters of the form `lower .. upper` are not expressions as a whole and must be handled separately.

* An indexing paramter of the form `lower .. upper`
is rewritten as `object.opStaticSlice!(i, lower, upper)`,
where `i` is the 0-based position of the slice in the square brackets.
* If this rewrite fails and the indexing expression is one-dimensional, the paramter
is rewritten as `object.opStaticSlice!(lower, upper)`,
i. e. without the position in the exprssion.
* If this rewrite also fails, the parameter is rewritten as `opSlice!i(lower, upper)`,
where `i` is the 0-based index of the slice in the square brackets.
(This behavior is currently part of the D Programming Language semantics and only mentioned for completeness.)
* If this rewrite fails and the indexing expression is one-dimensional, that part
is rewritten as `object.opSlice(lower, upper)`.
(This behavior is currently part of the D Programming Language semantics and only mentioned for completeness.)
* Otherwise, it is an error.

Although not strictly necessary, this DIP proposes to enforce that the template arguments of the lowering
member templates can only bind to template value parameters or to `alias` parameters as values,
i. e. to explicitly exclude types or aliases that do not refer to values as indexing arguments.

The DIP therefore proposes to rewrite any template argument `arg` to the static indexing operators to `cast(typeof(arg)) arg`
instead of plainly inserting them.

The rationale for that are corner cases where types and expressions could be conflated.
Consider a user-defined type `T` with the `static` member function `opIndex` declared `static R opIndex();`
(where `R` is an arbitrary type with no particular relevance to the example).
In a type context, `T[]` refers to the type of slices of `T`,
but in an expression context, it rewrites to `T.opIndex()` which has the type `R`.
Therefore, in this ambiguous context (usage as a template parameter), e. g. the expression `object[ T[] ]`,
this rule enforces the rewrite to `object.opStaticIndex!(T.opIndex())` and does not allow `object.opStaticIndex!(T[])`.
This explicit rule is necessary, since in the current state of the D Programming Language semantics,
if the expression `T[]` can be bound to a type parameter or a value parameter (even of exactly matching type),
the type parameter is preferred.
However, using e. g. `cast(typeof(T[])) T[]` enforces that the value parameter version is used.

It should be mentioned that the Language Maintainers could decide to only perfer the template value parameter
rewrite, but allow the binding of indxing parameters to other kinds of template parameter if the rewirites for value parameters fail.
The author of the DIP believes that this – although it would make static indexing more powerful – would also make it even more prone to operator abuse.
Secondly, such behavior could be allowed later if found useful in enough relevant cases.

Since static indexing operators need not be function templates but can be alias templates and therefore result in entities
that can be used in type contexts,
it might be of value to allow the rewrites in type contexts.
For static indexing to work in type contexts,
the grammar of the D Programming Language must be amended by the following clauses:

```diff
    QualifiedIdentifier:
        Identifier
        Identifier . QualifiedIdentifier
        TemplateInstance
        TemplateInstance . QualifiedIdentifier
        Identifier [ AssignExpression ]
        Identifier [ AssignExpression ] . QualifiedIdentifier
+       Identifier [ AssignExpression .. AssignExpression ]
+       Identifier [ AssignExpression .. AssignExpression ] . QualifiedIdentifier
```

In the current form of the grammar, in a type context, slicing expressions can only occur as the last part of a type (or somwehere inside).
This is reasonable because the slicing expression only makes sense if the part before it is a compile-time sequence of types
and such a sequence does not have members that are types, so it cannot maninfully be followed by `.` and an identifier.
By this proposal, this would no longer be the case anymore.
The same way indexing like `T[2]` can maninfully be followed by `.` and an identifier, this becomes also true for slicing expressions.

In particular, `Ts[1 .. 3]` may rewrite to `Ts.opStaticSlice!(1, 3)` which can be an alias to an aggregate with aribtrary members, including types.

Allowing static indexing operators in type contexts is not strictly necessary for the goal
of the DIP, but the rewrites would be expected by any user introduced to the concept of static indexing
and experienced with `alias` and templates in general.

### Optional Additional Proposal 1: Static Dollar

Additionally and optionally, the DIP proposes to add the compiler-recognized member `opStaticDollar`.
It is expected to evaluate to a compile-time constant.

If the symbol `$` occurs in an indexing expression, it is rewritten as follows:
* The symbol `$` is rewritten as `object.opStaticDollar!i`
where `i` is the 0-based index of the parameter containing the `$` in the square brackets.
* If this rewrite fails and the indexing expression is one-dimensional, the symbol `$`
is rewritten as `object.opStaticDollar!()`,
and as a second try `object.opStaticDollar`, i. e. without template instantiation syntax.
* If this rewrite fails, too, rewrites to `opDollar` are tried as per the current-state D Language Specification.

Additionally, the value of `opStaticDollar!i` (or `opStaticDollar` in the fallback, one-dimensional-only case)
must be a compile-time constant.
As per the current-state D Language Specification, this additional reqirement is not enforced for `opDollar`.

The author did not include `opStaticDollar` in the main proposal because it is not necessary for the goal of the DIP.
The only difference between `opDollar` and `opStaticDollar` would be that,
if only `opStaticDollar` is present,
when `$` is used in a indexing expression, there is a guarentee that it lowers to a compile-time constant,
while in the other case, the evaluation of `$` may have the costs of any other function call.
The case that both are present, but `opStaticDollar` does not compile, would likely be unusual and a bug.
The author recommends to make it an error if the rewrite to `opStaticDollar` does not compile if the member is present.
The case where both are present and compile would lead to `opStaticDollar` shadowing `opDollar` completly.
While it can stem from the programmer not understandig the rewriting,
it could also stem from the programmer supplying `opStaticDollar` and some mixin template supplying `opDollar`.

Also, for the user, it could be unclear that `opStaticDollar` would be used for rewriting `$` when the rest of the indexing expression
rewrites to dynamic indexing operators (because values that are not compile-time constants are involved).
But this follows the principle that the decision for static or dynamic indexing operators is orhtogonal throughout the expression:
When both sides of a slice indexing paramter of the form `lower .. upper` are compile-time constants,
the rewrite with `opStaticSlice` will be tried even if the rest of the indexing contains values that are not compile-time constants.
In this interpretation, the rewriting rules of `$` perfectly align.

### Optional Additional Proposal 2: Full Set of Dynamic Rewrites

Current state dynamic indexing operators rewrite the expressions of the form `object[]` and `object[lower .. upper]` to `object.opSlice()` and `object.opSlice(lower, upper)` respectively,
with appropriate hadling of unary operators and binary assignment operators and assignment
if the rewrites using `opIndex` fail.

Additionally and optionally, the DIP proposes to add the compiler-recognized members
* `opStaticSliceUnary`,
* `opStaticSliceAssign`, and
* `opStaticSliceOpAssign`.

Note that `opStaticSlice` is only absent from the list because the main proposal already proposes its addition.

The aforementioned list of static indexing rewrites would be ammended by:

* The expression `op object[]` or `op object[lower .. upper]`,
where `op` is an overloadable unary index operator,
is rewritten as `object.opStaticSliceUnary!op` or `object.opStaticSliceUnary!(op, lower, upper)` respectively.
* The expression `obj[] op= rhs` or `obj[lower .. upper] op= rhs`,
where `op` is an overloadable binary index operator,
is rewritten as `object.opStaticSliceUnary!op(rhs)` or `object.opStaticSliceUnary!(op, lower, upper)(rhs)` respectively.
* The expression `object[lower .. upper] = rhs`
is rewritten as `object.opStaticSliceAssign!(lower, upper)(rhs)`.

Notably absent is the rewrite of `object[] = rhs`.
This is due to the absence of a respective rewrite for dynamic indexing operators.

The author did not include this optional proposal in the main proposal because it is not necessary for the goal of the DIP.
While in the dynamic operator case, rationale of the analoguous rewrites is backwards compatibility,
as they are not deprecated,
they can be used for a cleaner code when one-dimensional indexing and slicing is implemented.
The main argument for this additional proposal is that users can convert dynamic and static indexing into each other
by merely moving template to function parameters and adding to or removing “Static” from the member name.
If these rewrites were missing, for the backwards compatibility slicing operator, more complicated work would have to be done.
Also, understanding static indexing would require the programmer to remember that parts of dynamic rewrites do not have direct static equivalents.

### Optional Additional Proposals 3 and 4: Expansion Expression and Expansion Operator

<!--- WORK IN PROGRESS --->

Additionally and optionally, the DIP proposes that for an Expression `expr` to make `expr...` a valid Expression, too.
Then, `expr...` is called an *expansion expression*.

This new kind of expression is subject to the Optional Additional Proposals 3 and 4 which are separate apart from that.

For the `...` token be accepted in expression contexts, the grammar must be amended:
```diff
    PostfixExpression:
        PrimaryExpression
        PostfixExpression . Identifier
        PostfixExpression . TemplateInstance
        PostfixExpression . NewExpression
        PostfixExpression ++
        PostfixExpression --
+       PostfixExpression ...
        PostfixExpression ( ArgumentListopt )
        TypeCtorsopt BasicType ( ArgumentListopt )
        IndexExpression
        SliceExpression
```

Note that current expressions containing `...` are an error:
The only place where they could be legal in the current state of the D Programming Language
is when in a Slice of a SliceExpression, the upper bound expression starts with a PrimaryExpression of the form `. Identifier`.
An example for such an expression is `object[lower .. .upper]`.
Then the space after the two dots in the Slice cannot be omitted:
Whenever three dots are next to each other, as the lexer is eager for the `...` token, such an expression, e. g. `object[lower...upper]`, is invalid syntax.

Therefore, adding the aforementioned kind of PostfixExpression is a mere addition.

Indexing is not only present in expressions but also in type contexts, e. g. parameter lists of functions.
Therefore, the DIP suggests rewriting `Types...` to `Types[0]` through `Types[length - 1]`,
where `length` being replaced by `Types.length`, `Types.opStaticDollar`, or `Types.opDollar` (cf. above).
Here, `Types` can be, but not necessarily is, a compile-time sequence of types;
it can be an alias of anything with the members necessary for static indexing.

For the `...` token be accepted in type contexts, the grammar must be amended:
```diff
    TypeSuffix:
        *
        [ ]
        [ AssignExpression ]
        [ AssignExpression .. AssignExpression ]
+       ...
        [ Type ]
        delegate Parameters MemberFunctionAttributes[opt]
        function Parameters FunctionAttributes[opt]
```

The type part is independent of the expression part and only included
as it could be surprising that expanding works in expressions,
but not in a type context, although the procedure could be done there.

In contrast to expression contexts, in type contexts, the `...` token is already in use:
* The template parameter sequence is unaffected.
* The D-style variadic functions[⁽⁹⁾] are unaffected.
* The type-safe variadic functions[⁽¹⁰⁾] are unaffected
if the parameter is named.
* The type-safe variadic functions are also unaffected if the type of the variadic argument does not have members necessary to expand.

An example for the last point is this:
```d
void f(Type...);
```
If `Type` is a compile-time sequence of types, this is currently equivalent to listing the types one by one;
this is not affected by any of the following proposals.
Currently, when `Type` is an alias for an array type (both, dynamic array or static array) or a class type,
by the current semantics of the language, the function takes an argument of that type,
or, in the dynamic array case, any number of arguments implicitly convertible to the array element type,
or, in the static array case, as many arguments implicitly convertible to the array element type as the length of the array specifies,
or, in the class case, any arguments that a constructor of the class takes.

To summarize, if a _class_ (but not a struct or union)[⁽¹⁰⁾]
* defines static members for static indexing,
* is the type of the last parameter of a type-safe variadic function, and
* that last function parameter is not named,

by optional additional proposals 3 and 4, those function definition changes meaning.
However, exisiting code is extremely unlikely to contain the compiler-recognized members for static indexing.

#### Optional Additional Proposal 3: Expansion Expressions

If the type of `expr` has a compile-time known `length` or `opDollar` member
or, in the case that the optional proposal 1 is accepted, has the (required by the proposal: compile-time known) member `opStaticDollar`
and also the static indexing expressions `expr[0]` to `expr[length - 1]` compile,
where `length` is `expr.length`, `expr.opStaticDollar`, or `expr.opDollar`
(the first one of them that compiles expecting that they are the same value),
`expr...` rewrites to the compile-time sequence `expr[0]`, `expr[1]`, …, `expr[length - 1]`.
Note that the requirements above do not implay that all of these expressions actually compile,
but that is necessary for the `...` rewrite into a sequence to be successful.

Note that
* the result of an expansion is *not* a comma expression, but a compile-time sequence of aliases.
* if the sequence includes values, those are not required to be compile-time constants.
* in contrast to C++’s `...` in expressions, this DIP does not propose expansion be forwarded into the expression.

Forwaring into the expression means that if expandable expressions are combined and followed by the expansion operator,
e. g. `(expA + expB + 1)...`,
and the expression before the expansion operator has no type (i. e. `is(typeof(expr))` is `false`)
or the resulting type does not overload the static indexing operators,
the whole expression would be lowered to a sequence of piece-wise expansions,
e. g. to `(exp1[0] + exp2[0] + 1)`, …, `(exp1[$ - 1] + exp2[$ - 1] + 1)`.
Forwaring into expressions is subject to a DIP in draft state at the time of this writing.

The author believes that such a behavior would be suprising to programmers that have no knowledge of C++’s expansion operator.
Also, in contrast to C++, in D one can mimic that behavior using proposed and existing language features at least to some extent.

The author did not include the expansion expression in the main proposal because it is not necessary for the goal of the DIP.
However, for implementing custom tuple-like types, it may be useful to have a simple but visible way to expand the objects,
e. g. to pass the elements of the tuple as individual arguments as function parameters or template value parameters.

#### Optional Additional Proposal 4: Expansion Operator

Additionally and optionally, the DIP proposes to add `opExpand` to the list of compiler-recognized member names.
The expression `expr...` is rewritten to `expr.opExpand!()` and as a second try `object.opExpand`,
i. e. without template instantiation syntax,
if the type of `expr` defines that member.

Additionally, in a type context, if `Ts` is an identifier, `Ts...` is rewritten to
`Ts.opExpand!()` or `Ts.opExpand`, respectively, if `Ts` has that member.
Otherwise, rewrites introduced by the optional additional proposal 3 may be applied.

The author did not include the expansion operator in the main proposal because it is not necessary for the goal of the DIP.
Although not part of indexing, the considered the addition as it is linked to indexing.
Many constructs using static indexing could also make use of the expansion operator.
In many cases, the optional proposal 3 would suffice, but a user-implemented expansion operator
can hook the semantics in any way the programmer whishes.

<!--
### Optional Additional Proposal 5: Construction for Structs and Unions

Additionally and optionally, the DIP proposes to add support for variadic typesafe functions based on structs.

Currently, only dynamic and static arrays and classes are supported.[⁽¹⁰⁾]
Struct and union types could be supported the same way as classes are:
When the argument in place is not convertible to the parameter type,
an implicit constructor call is issued (the only difference to classes is omitting `new`).

This would make it not only possible to have tuple-like types replace static arrays in that place,
it also allows implicit construction in general.

```d
alias Tup = TupleLike!(int, string);
void f(Tup tup...) { }

f(1, "two"); // rewrites to f(Tup(1, "two"));
```

The author believes that this is entirely different from implicit constructor calls like non-`explicit` constructors in C++,
since the construction is made visible in the signature of the function.

Together with Optional Additional Proposal 3 and 4, this enables object construction parameters to be encapsulated in
tuple-like structures and being expanded where an object to be constructed is being used.

```d

```
-->

## Examples

### Closed Tuples

Static indexing could be used to implement ecapsulated tuples which do not auto expand (in contrast to Phobos’ `Tuple`[⁽²⁾]).
* The entries can be read (by copy) with the syntax `tuple[index]`, but not (re)assigned after construction.
* Slice expressions of the tuple are sub-tuples, typed as a template instance of the tuple template.

```D
struct ReadOnlyTuple(Ts...)
{
    private Ts values;

    Ts[i] opStaticIndex(size_t i)() { return values[i]; }

    static struct Slice { size_t lowerBound, upperBound; }

    // Note: This fails, but should work. TODO: File bug report.
    alias opSlice = Slice;

    template opStaticIndex(Slice slice)
    {
        alias SubTuple = ReadOnlyTuple!(Ts[slice.lowerBound .. slice.upperBound]);
        SubTuple opStaticIndex()
        {
            return SubTuple(values[slice.lowerBound .. slice.upperBound]);
        }
    }
}

unittest
{
    alias T4 = ReadOnlyTuple!(int, string, char, ulong);
    T4 tup4 = T4(1, "foo", 'c', ulong.max - 1);
    assert(tup4[0] == 1); // rewrites to assert(tup4.opStaticIndex!(0) == 1);
    alias T2 = ReadOnlyTuple!(string, char);
    auto tup2 = tup4[1 .. 3]; // rewrites to auto tup2 = tup4.opStaticIndex!(tup4.opSlice(1, 3));
    static assert(is(typeof(tup2) == T2));
    assert(tup2[0] == tup4[1]); // rewrites to assert(tup2.opStaticIndex!0 == tup4.opStaticIndex!1);
    assert(tup2[1] == tup4[2]); // rewrites to assert(tup2.opStaticIndex!1 == tup4.opStaticIndex!2);
}
```

### Tuples with Slice Operator

Tuples can have a varying degree of homo-/heterogeneity:
* It can be perfectly homogeneous, i. e. all the entries have exactly the same type, e. g. a tuple of `int` and `int`.
* It can be qualifier homogeneous, i. e. the unqualified types of the entries are exactly the same type and there is a (common) qualifier that all entry types are convertible to,
e. g. a tuple of `int` and `immutable(int)` (the common qualifier being `const`),
or else a tuple of `immutable(int)` and `shared(int)` (the common qualifier being `shared const` since `immutable` objects are implicitly `shared`);
an interesting counter-example would be `int` and `shared(int)` which lack a common qualifier.[⁽³⁾]
* It can be class homogeneous in the sense that all the entries are of class type and have a common base; as a last resort, that base class might be `Object`, but it may be more concrete.
* It can be convertible homogeneous, i. e. there is a type that all entries can be converted to.
The selection of the common type could allow user-defined conversions, e. g. setting the common type of `int` and `uint` to `long`.
* It is heterogeneous if it is neither of the above.

The homogeneity properties can be made use of via Design by Introspection:[⁽⁴⁾]
* Perfectly homogeneous tuples resemble static arrays and can be converted to a slice of the unique underlying type.
They can be indexed dynamically and `opIndex` can return by reference.
* Qualifier homogeneous tuples can be sliced, too: The types hava a common type qualifier using implicit qualifier conversions[⁽³⁾] and that type is the underlying type of that slice.
The difference to perfect homogeneity being that the type returnded by `opIndex` and `opStaticIndex` need not be the same.

```D
import sophisticated.tuple : Tuple;
import pack.a : A;

alias A2 = Tuple!(A, immutable(A));
A2 a2 = A2(A(1), immutable(A)(2));
// A2 being qualifier homogeneous, provides a2.opIndex() returning const(A)[].
auto slice = a2[]; // rewrites to auto slice = a2.opIndex();
static assert(is(typeof(slice) == const(A)[]));

// A2 being qualifier homogeneous, provides opIndex(size_t index) returning const(A) by reference.

inout(A) f(ref inout(A) value);

size_t i = userInput!size_t();
auto x = f(a2[i]); // rewrites to auto x = f(a2.opIndex(i));
```

* Class homogeneous tuples cannot be iterated by mutable reference,
but sliced returning `const(Base)[]` or `const(shared(Base))[]` depending on `shared` and iterated by `const` reference using `opApply`;
occasionally, an improved implementation could potentially yield better types like `const(inout(shared(Base)))[]`.
* Convertible homogeneous tuples cannot be sliced in general, but be indexed by value and iterated by value using `opApply`
since the conversion may be destructive and/or change the size of the object.

```D
import sophisticated.tuple : Tuple;
import sophisticated.tratis : CommonType;
import pack.a : A;
import pack.b : B;
import pack.c : C;

static assert(is(CommonType!(A, B) == C));

alias AB = Tuple!(A, B);

// AB being convertible homogeneous with common type C provides opIndex(size_t i), which converts the
// i-th value at run-time to the common type C, and opApply that walks the iteration variable
// through the conversions of the entries of the tuple to the common type C.

AB ab = AB(A.init, B.init);
foreach (i, c; ab) // non-static loop
{
    static assert(is(typeof(c) == C));
    if (i == 0) assert(ab[0] == ab[i]); // rewrites to assert(ab.opStaticIndex[0] == ab.opIndex[i]);
    if (i == 1) assert(ab[1] == ab[i]); // rewrites to assert(ab.opStaticIndex[1] == ab.opIndex[i]);
    // The last two asserts need not actually hold (cf. conversion from ulong.max to float),
    // but if the conversion from A and B to C is lossless (e.g. the conversion from int to long),
    // it is reasonable to expect them to hold.
}
// As the conversions of the entries are intermediate values, `ref` iteration is not possible.

size_t i = userInput!size_t();
static assert(is(typeof(ab[i]) == C));

void f(ref C value); //1
void f(    C value); //2
auto x = f(ab[i]); // rewrites to auto x = f(ab.opIndex(i)); and calls //2
```

All of this is not possible in the current state of the D Programming Language,
because defining `opIndex` makes the compiler not rewrite the tuple in terms of its alias this.
This is true even if the call to `opIndex` does not compile.

### Compile-time Random-access Ranges

One can implement some kind of compile-time version of Phobos’ `iota`[⁽⁵⁾] and other ranges.
To the outside oberserver, these seem to contain values, but the values are caluclated
from internal state without creating the values beforehand.

```D
template iota(size_t start, size_t end, size_t step)
if (start <= end && step > 0)
{
    template opStaticIndex(size_t index)
    {
        static assert (start + index * step < end, "static index out of range");
        enum opStaticIndex = start + index * step;
    }

    template opStaticSlice(size_t lower, size_t upper)
    if (lower <= upper && upper <= length)
    {
        alias opStaticSlice = iota!(start + lower * step, start + upper * step, step);
    }

    enum size_t length = (end - start) / step;
    alias opDollar = length;
}

alias i = iota!(1, 10 + 1, 2); // 1, 3, 5, 7, 9

static assert(i[0] == 1); // rewrites to static assert(i.opStaticIndex!0 == 1);
static assert(i[1] == 3); // rewrites to static assert(i.opStaticIndex!1 == 3);

int[] array = [ i... ]; // rewrites to int[] array = [ i[0], i[1], i[2], i[3], i[4] ];
```

It could be argued that this semantics can be implemented easily in the current form of the language
and that the proposed static indexing syntax is mere a readability concern,
but syntax is relevant in a meta-programming context.

<!-- NEEDS FURTHER INVESTIGATION -->
<!--
enum bool isCompileTimeConstant(alias value) = is(typeof(value)) &&
    __traits(compiles, {
        enum e = value;
    });
auto format(alias fmt, Ts...)(auto ref Ts args)
{
    import std.format : format;
    import std.functional : forward;
    static if (isCompileTimeConstant!fmt && is(typeof(fmt) : const(char)[]))
    {
        return format!fmt(forward!args);
    }
    else
    {
        return format(fmt, forward!args);
    }
}
void main()
{
    import std.stdio;
    write("fmt: ");
    auto fmt = readln;
    format!fmt(1, "zwei").writeln;
    static assert(!__traits(compiles, format!"%d  %d"(1, "zwei")));
    static assert(!__traits(compiles, {
            static import std.format;
            auto r = std.format.format!fmt(1, "zwei");
        }));
}
-->
<!--
### Simulate Compile-time Aware Function Parameters

Utilizing `static` operator members, different code could be generated depending
whether parameters are compile-time constants or not.
The most prominent example that could profit from that is the function `format`.
Also, users of `regex` and `ctRegex` would have a uniform syntacitcal expression, `regex[pattern]`,
that decides on compile-time availableness of `pattern` which regex engine to use
(the compile-time or the run-time one).
In current D, the caller must use different syntax to make the compiler check whether format specifiers
and argument types match: `format!(formatString)(arguments)` versus `format(formatString, arguments)`.
This differnece is minor, and the user mostly would want the checks in every case where it is possible.
The same holds true for regular expressions. Most of the time, the user would want to get the best one.
This proposal does not inhibit the user from manually deciding for a compile-time or run-time option
using the currently existing forms. In meta-programming contexts, the format or pattern might even
be given by an alias, so the template using the index syntax need not do a case distinction
on the compile-time availableness of the parameter.

In the current state of D, there is no construct that decides on function parameters being compile-time constants. (THIS IS WRONG!)
A function that does the checks when possible and otherwise emits run-time checks cannot be implemented.

Here is an example how this can be implemented.

```D
struct format
{
    template opStaticIndex(string fmt)
    {
        static string opStaticIndex(Ts...)(Ts arguments)
        {
            import std.format : format;
            return format!fmt(arguments);
        }
    }

    static auto opIndex(string fmt)
    {
        return Formatter(fmt);
    }

    struct Formatter
    {
        string fmt;
        this(string fmt) { this.fmt = fmt; }

        string opCall(Ts...)(Ts arguments)
        {
            import std.format : format;
            return format(fmt, arguments);
        }
    }
}

unittest
{
    string fmt = "%s";
    auto s = format[fmt](1); // rewritten auto s = format.opIndex(fmt).opCall(1)
    static assert(is(typeof(s) == string));
    assert(s == "1", "s == '" ~ s ~ "'");
}

unittest
{
    enum string fmt = "%s";
    enum s = format[fmt](1); // rewritten enum s = format.opStaticIndex!fmt(1);
    static assert(is(typeof(s) == string));
    static assert(s == "1");
}
```

It can be argued that this usage of the static and dynamic indexing operators is a form of abuse of operator overloading.
If the changes proposed by this DIP are implemented, the community will decide if such code
is considered abuse of operator overloading and should therefore be discuraged (in most cases) or if it becomes a pattern
to solve this kind of problem.
-->

### Possibilties in Popular DUB-Libraries

*The author hereby requests ideas any contributor of popular DUB repositories might have.
The author expects that there are many optimizations possible especially in Mir.*

## Alternatives

### Current State

Currently, the only way to mimic static indexing is using alias this to a compile-time sequence (generated e. g. by Phobos’ `AliasSeq` template[⁽⁶⁾]).
In Phobos, `Tuple` does exactly that.[⁽²⁾]
This has several limitations:

1. While it is possible to have alias this to the sequence and the compiler-recognized member `opIndex` in place,
the index syntax is not forwarded to the alias this’d sequence anymore
even if the rewrite with `opIndex` fails.
Phobos’ `Tuple` does not overload `opIndex` and therefore does not suffer from this problem.[⁽²⁾]
2. Even if 1. were resolved, the result of a static slice (e. g. `object[l .. u]`) would be a alias sequence.
There is no way to hook in and make it evaluate to something different than a sequence, e. g. a user-defined type.
Consequently, Phobos’ `Tuple` when sliced, does not return an appropriate `Tuple` but a sequence.[⁽²⁾]
This is not the case for anything else that can be indexed and sliced.
3. The sequence cannot be encapsulated or protected from direct outside modifications.
It is fully exposed without any possibility to manage access to it.
4. The complete sequence must exist in the first place.
It might be desirable to pretend the existence of a sequence without storing an actual sequence (cf. Phobos’ `iota`[⁽⁵⁾]).
5. As the place of alias this is already taken, one cannot alias this to something different.
It is a many-year-long standig issue that multiple alias this is not implemented.

The whole sequence approach can only be used for indexing with exactly one parameter.
Multidimensional (including 0-dimensional) indexing is not possible.

Without indexing syntax, it is necessary to use helper function templates, that take
the indexing parameters as template parameters,
e. g. like `tuple.index!1` or `tuple.slice!(lBound, uBound)`.

The author believes that using those helper templates is less readable than indexing expressions.
More objectively, it does not play well in meta-programming contexts,
where one would like to not have to distinguish the cases of user-defined types,
static arrays and compile-time sequences
the same way, meta-programming code does not have to distinguish
dynamic arrays and user-defined random-access ranges when indexing them.
The current solution is to ignore tuple-like types or to special-case specific ones.

### Specific Syntax

If the Language Maintainers decide that overloading indexing syntax is not desirable,
the DIP author proposes to use the syntax `object![indices]` (note the exclamation mark) for static indexing.
This syntax is is currently an error as explicit template instantiation needs parentheses after the exclamation mark
or a single one-token argument, but anything starting with an opening bracket is not a one-token argument.[⁽⁷⁾]
Therefore, the alternatively proposed syntax would be a mere addition to the language.
Accordingly, indexing compile-time sequences should also be possible to do with the syntax `sequence![index]`
to cater to meta-programming contexts.

In this case, the grammar of the D Programming Language has to be expanded.
Section 10.22 of the specification[⁽⁸⁾] could be amended as follows:
```diff
    SliceExpression:
        PostfixExpression [ ]
        PostfixExpression [ Slice ,[opt] ]
+       PostfixExpression ! [ ]
+       PostfixExpression ! [ Slice ,[opt] ]
    Slice:
        AssignExpression
        AssignExpression , Slice
        AssignExpression .. AssignExpression
        AssignExpression .. AssignExpression , Slice
```
The author is convinced that distinguishing static and dynamic indexing is an unnecessary inconveniance which
takes away possibilities and lacks the necessary justifications,
the only true justification being that static and dynamic indexing must be distinguishable.
That could be argued against that it is not distinguishable in the current state of the language and
that it never bothered D programmers enouth to propose changes.
On the other hand, there are constructs in the standard library that purposefully introduce a name that,
depending on template parameters, is type or a value. In most cases D programmers are used to compile-time and
run-time things looking the same syntactically.

### Using Dynamic Indexing Member Functions for Both

It was suggested to the author that the compiler-recognized members for dynamic indexing could be used for both,
dynamic and static indexing. The author rejected this suggestion because such an addition might change what current code intends,
and in corner cases, silently changes behavior.
While still theoretically possible, adding new compiler-recognized members is less likely to introduce silent changes (cf. [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)).

An illustration of such a case is this:
```D
struct Example
{
    int opIndex(size_t i = 0)(size_t[] xs...) { return i; }
}
```
Let `example` be of type `Example` and given the expression `example[1]`,
it rewrites to `example.opIndex(1)` which compiles and returns `0`;
but if the rewrite `example.opIndex!1()` were tried first,
it succeeds and returns `1`.

While this example is somewhat artificial,
it still demonstrates that breakage and silent change of meaning is possible in less fringe code.

### Compile-time Function Parameters

If some way to discriminate function parameters for their compile-time availableness were implemented,
e. g. by adding a function parameter storage class (here: `enum`),
the functionality would be implementable using the currently available compiler-recognized members for operator overloading.

Example:
```D
auto opIndex(enum size_t index) { ... } // A
auto opIndex(     size_t index) { ... } // B
```
Overload A catches all uses of `obj[index]` (i. e. `obj.opIndex(index)`) where `index` is a compile-time constant.
Overload B catches all remaining legal uses of `obj[index]`, i. e. where `index` is a runtime value.

Discriminating function parameters for their compile-time availableness is somewhat similar to
the distinction of value categories (lvalue vs. rvalue), i. e. if the parameter can be bound to `ref` or not.

The author believes that adding such a storage class is generally useful, but a much greater change of the language as
it affects the mechanics of function overloading and makes reasoning about code more difficult.
Walter Bright has a record of opposing porposed changes to the overloading rules.
(Alt least changes, that make them more difficult, which is the case for almost all of them.)
Detailing out, what such a change would exactly entail, is beyond the scope of this DIP.
The author only mentions this alternative as this might encourage its evaluation by someone else.
Also, this DIP would likely be superseded if such a language change be accepted.

## Breaking Changes and Deprecations

The changes of the main proposal and the optional propsals are purely additional.
They only affect user-defined types that have members named the same as the newly introduced compiler-recognized member names
which the author presumes to be unlikely because of the length and pattern (`opSomething`) of the names.
Such code would only be affected if indexing syntax is used on those types;
those would now rewrite using the compiler-recognized members.
Refactoring the conflicting member names will preserve the behavior.

## References

<!--- List of references --->

1.  [D Language Specification on Operator Overloading](https://dlang.org/spec/operatoroverloading.html)
2.  [D Library Documentation of std.typecons.Tuple](https://dlang.org/library/std/typecons/tuple.html)
3.  [D Language Specification on Implicit Qualifier Conversions](https://dlang.org/spec/const3.html#implicit_qualifier_conversions)
4.  [*Design by Introspection* by Andrei Alexandrescu, talk at the D Language Conference 2017](https://youtu.be/29h6jGtZD-U)
5.  [D Library Documentation of std.range.iota](https://dlang.org/phobos/std_range.html#.iota)
6.  [D Library Documentation of std.meta.AliasSeq](https://dlang.org/library/std/meta/alias_seq.html)
7.  [D Language Specification on Explicit Template Instantiation](https://dlang.org/spec/template.html#explicit_tmp_instantiation)
8.  [D Language Specification on Slice Expressions](https://dlang.org/spec/expression.html#slice_expressions)
9.  [D Language Specification on D-Style Variadic Functions]()
10. [D Language Specification on Typesafe Variadic Functions](https://dlang.org/spec/function.html#typesafe_variadic_functions)
11. [Issue 16302 in Dlang’s Issue Tracking System](https://issues.dlang.org/show_bug.cgi?id=16302)

<!--- Markdown reference definitions --->

[⁽¹⁾]: #references "D Language Specification on Operator Overloading"
[⁽²⁾]: #references "D Library Documentation on std.typecons.Tuple"
[⁽³⁾]: #references "D Language Specification on Implicit Qualifier Conversions"
[⁽⁴⁾]: #references "D Language Conference 2017, Talk by Andrei Alexandrescu, Design by Introspection"
[⁽⁵⁾]: #references "D Library Documentation on std.range.iota"
[⁽⁶⁾]: #references "D Library Documentation on std.meta.AliasSeq"
[⁽⁷⁾]: #references "D Language Specification on Explicit Template Instantiation"
[⁽⁸⁾]: #references "D Language Specification on Slice Expressions"
[⁽⁹⁾]: #references "D Language Specification on D-Style Variadic Functions"
[⁽¹⁰⁾]: #references "D Language Specification on Typesafe Variadic Functions"
[⁽¹¹⁾]: #references "Issue 16302 in Dlang’s Issue Tracking System"

## Copyright & License

Copyright © 2019 – 2020 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
