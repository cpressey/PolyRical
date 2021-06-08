PolyRical
=========

PolyRical is a successor language to [SixtyPical][].  It explores a few (but by no
means all) of the avenues mentioned in the [Future directions for SixtyPical][] document.

Like SixtyPical, it is a very low-level language that nonetheless supports advanced
forms of static analysis.  It aims to be based on a cleaner, more general theory of
operation than SixtyPical, and thus (ideally speaking) able to target architectures
other than the 6502.

This document is still at the **design stage**.  No code has been written, and all of
the decisions described here are subject to change.

[SixtyPical]: https://catseye.tc/node/SixtyPical
[Future directions for SixtyPical]: https://gist.github.com/cpressey/f35e104b3e3cf555824aa2b4d15ea858

Motivating example
------------------

A PolyRical program consists of pragmas, templates, and declarations.  Typically one
would import a set of templates from a library using a pragma, but to convey the
flavour of the the language, here is a self-contained example program:

    word[8] register A
    word[8] location score @ 0xc000
    
    template load(out A, word[8] value val) {
        0xA9 [val]                                  /* LDA immediate */
    }
    
    template store(in A, out word[8] location dest) {
        0x8D [<dest] [>dest]                        /* STA absolute */
    }
    
    routine(out score, trash A) main {
        load(A, 0)
        store(A, score)
    }

Note that this is enough information to consider the version of `main` show above
to be valid, and to (almost) produce a machine-language program for it, while rejecting

    routine(out score, trash A) main {
        store(A, score)
    }

with a message such as `'store' expects 'A' to be meaningful but in 'main' it is not necessarily so`,
and also rejecting

    routine(out score, trash A) main {
        load(A, 0)
    }

with a message such as `'main' does not give a meaningful value to 'score'`.

In more detail
--------------

A declaration consists of a type, a role, a name, an optional initializer, and
an optional location.  Some combinations are not valid — for instance, any declaration
with the role `value` must include the initializer.

Declarations declare global variables, constants, and routines.  A routine is
just a constant of `routine` type.

Each routine type is parameterized with a set of declarations and constraints on
those declarations.

There are no parameters to a routine, and no local variables.  All variables are
global.  But each routine must conform to the constraints that its type imposes on
the declarations on the program.  So, if the type declares that the variable `A`
will be given a meaningful value by the routine, then the routine must give it a
meaningful value, or the implementation should signal an error condition.

Templates, too, have such constraints.  Unlike routines, templates do have a list
of parameters.  Also unlike routines, the constraints declared by a template are
not checked by the implementation (often they would not be checkable, as templates
provide machine-level details for implementing an operation).  But they are
propagated to the rest of the program.  Most programmers would use a library of
templates instead of writing their own.

A routine body consists of a list of template applications.  The template
definition used in a template application is retrieved based on the types and
roles of the declarations given as parameters.  This is like method overloading in
languages such as Java.  Actually, it goes further, in that a template definition
can list a particular declaration, rather than just a type and role, as one of its
parameters.  This definition will be selected when this exact parameter is used.
The `load` and `store` templates in the above example demonstrate this for the
declaration `A`.

### Meaningful values

The most prominent property of declarations that PolyRical tracks is _meaningfulness_.
This is very similar to how many C compilers track _defineness_ of variables, and
are able to warn the user if the code uses a variable that is not defined, or may not
be defined in all cases.

Meaningfulness is controlled by three constraints on each routine type and on each
template:

*   `in` asserts that a declaration must be meaningful before the routine or template is applied
*   `out` asserts that a declaration will be meaningful after the routine or template is applied
*   `trash` asserts that a declaration will not be meaningful after the routine or template is applied

The meaningfulness of all other declarations is preserved when a routine or template
is applied.

In particular, `in` asserts the meaningfulness of a declaration during input, so
in the absence of `trash` on the same declaration, the declaration is assumed to
also be meaningful on output.

Beyond meaningfulness, other properties of declarations such as range, and
whether a routine is ever called, are trackable by symbolic execution.

### Data types

The most basic data type is the `word`, which, because we want the possibility
of targeting systems other than the 6502, is parameterized by its size.  An
8-bit byte is `word[8]` and a 16-bit word is `word[16]`.  Following this pattern,
a single-bit flag register is `word[1]`.

Routine types have already been discussed.

It is likely that array, index, and pointer types will also be available.  It is
also likely that index and pointer types can be restricted to pointing within a
a particular array.

### Roles

Along with a type, each declaration has a role.  There are two main roles, which
are `location` and `value`.  These are similar to the concepts of `lvalue` and
`rvalue` respectively, in languages such as C.  A declaration which is a
`location` supports the operation of having its address taken; it does not
directly support having its value set or read, but there may be machine
instructions which do this, and one or more templates defined which use them.
A `value`, on the other hand, does not support having its address taken, but
does directly support having its value taken.

Roles are considered when selecting a template for given actual parameters.
This prevents such things as trying to assign a value to another value; there
will typically be no such template with that signature.

Every literal constant is considered a `value`.

There are other, more specialized roles.  `register` is like `location` but
does not support having its address taken.

### Control flow

Static analysis is easy if all programs are entirely straight-line code — the
complications come up when branching occurs.

But branching is also an opportunity, because every time a branch occurs, the
analyzer has more information about what conditions must pertain in each branch.
(cf. flow typing)

For example, if we branch on the carry flag, we know that, in the code that we
branch to, the carry flag must always be set; and in the code where the branch
was failed to be taken, the carry flag must always be clear.

Ideally, we'd like to capture control flow in templates; templates could take
blocks as parameters, allowing the programmer to, for example, define a template
called `ifzero` that takes a block and generates the machine code for making a
test against zero, making a jump, and machine code for the entire block.  This
would allow us to implement control structures in an architecture-agnostic way.

Unfortunately, that also complicates analysis immensely.  So, while that is
a noble goal, we may stick to hard-coded control structures for the first few
versions.

One way to live with hard-coded control structures is to have templates that
are employed implicitly.  For example, the reason we said the motivating
example given above is only "almost" enough information to produce a machine-language
program, is that we haven't defined what the `main` routine should do when it's
finished.  Presumably it should return to its caller, but the program doesn't
provide a way to say that in machine language.  So we would need to define
an implicit template like

    template _return() {
        0x90                                    /* RTS */
    }

and the compiler would insert this at the end of each routine as necessary.
