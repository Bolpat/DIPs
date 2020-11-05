# Uniform Function Parameter Binding

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id)                                                     |
| RC#             | 0                                                               |
| Author:         | Quirin F. Schroll (q.schroll@gmail.com)                         |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |


## Abstract

This DIP solves the longstanding problem that it is not possible in D to specify the binding of l-value and r-value parameters alltogether.
It provides an easy to understand and easy to implement mechanism for that by redefining the sematics of `in` as a new parameter storage class.

This proposal is not about templates and `auto ref` at all.
See Rationale for why templates and `auto ref` don't suffice to solve the problem.

### Links

My reply to the question [`in` no longer same as `const ref`](https://forum.dlang.org/post/medovwjuykzpstnwbfyy@forum.dlang.org) as the inital point to this DIP.

Ali Çehreli's answer to [meaning of "auto ref const"?](https://forum.dlang.org/post/o3c341$2sqn$1@digitalmars.com).

Jonathan M Davis [answer](https://forum.dlang.org/post/o6626e$afu$1@digitalmars.com) pointed out why move constructors are not necessary in D,
see also [std.move](https://dlang.org/phobos/std_algorithm_mutation.html#.move).


## Description

An `in` parameter is implicitely `const` and `scope`.
Writing `const` or `scope` together with `in` is not an error; a compiler implementation may give a warning.
It is `return` iff
* the function is a template and `return` is inferred, or
* the parameter is annotated `return` manually.

The parameter then binds any r-value by move and any l-value by reference.


### Rationale

Assume a struct type `S` that is possibly expensive to copy but cheap to move, i.e. an array-type with unique ownership like C++'s `std::vector`s.
One wants to bind function parameters of that type most effectively with the most flexibility and without much extra lines of code.
This DIP proposes to change the meaning of `in` and `ref` as a parameter storage class completely.

If you want to bind a value of `S` just to read from it without care where the value came from, there is no simple one-overload way to archieve that in D.
To bind l-values cheap you need `ref` which disallows r-values today.
To bind r-values cheap you omit `ref` which copies    l-values today.
So you overload the function with one taking the copy and call the other overload with `ref`.

#### `auto ref` Functions

The possibility to make the function a template and use `auto ref` is not viable for virtual functions or when separating declaration and definition.
Therefore it is an incomplete solution to this problem.

### Use Cases

#### Typical ones
* To read from the instance where you don't care where it came from: `f(in ref const S)`.
* To manipulate the instance where you don't care where it came from: `f(in ref S)`.

The latter is ...

Specific

### Comparison with C++ 11

| C++             | D                | Semantics Penalty      |
|-----------------|------------------|------------------------|
| `f(S         )` | `f(          S)` | copy                   |
| `f(S const   )` | `f(    const S)` | copy, no mutate        |
| `f(S        &)` | `f(ref       S)` | no r-values            |
|        —        | `f(ref const S)` | no r-values, no mutate |
| `f(S const & )` | `f(  in      S)` | no mutate, (no alias)  |


### About `in` as a Shorthand

Using `in` as a shorthand for something as simple as `scope const` has been a bad design decision with subtle disadvantages:

1. One has to learn something without extra value; the abbreviated context is not that difficult and common that it'd need the abbreviation at all.
   The `scope` storage class even has no effect on many types.
2. Programming languages strongly tend to only define something with specific abilities.
   As an abbreviation, `in` obviously cannot have specific abilities.
   Beginners typically have problems with that.
3. `in` meant to be the opposite of `out`, but it is far away from that.


### Breaking changes / deprecation process

Provide a detailed analysis on how the proposed changes may affect existing
user code and a step-by-step explanation of the deprecation process which is
supposed to handle breakage in a non-intrusive manner. Changes that may break
user code and have no well-defined deprecation process have a minimal chance of
being approved.

### Examples

A more practical explanation of DIP semantics should be given by showing
several examples of its idiomatic application. Inspecting vivid code examples
is usually an easier & quicker way to evaluate the value of a proposal than
trying to reason about its abstract description.

## Copyright & License

Copyright (c) 2017 by Quirin F. Schroll.

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

Will contain comments / requests from language authors once review is complete,
filled out by the DIP manager - can be both inline and linking to external
document.
