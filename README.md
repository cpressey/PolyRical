PolyRical
=========

PolyRical is a successor language to [SixtyPical][].  It explores a few (but by no
means all) of the avenues mentioned in the [Further Directions for SixtyPical][] document.

Like SixtyPical, it is a very low-level language that nonetheless supports advanced
forms of static analysis.  It aims to be based on a cleaner, more general theory of
operation than SixtyPical, and thus (ideally speaking) able to target architectures
other than the 6502.

[SixtyPical]: https://catseye.tc/node/SixtyPical
[Further Directions for SixtyPical]: TK

### A motivating example

A PolyRical program consists of pragmas, templates, and declarations.  Typically one
would import a set of templates from a library using a pragma, but to convey the
flavour of the the language, here is a self-contained example program:

    word[8] register A
    word[8] location score @ 0xc000
    
    template(out A) load(A, word[8] value val) {
        0xA9 [val]                                  /* LDA immediate */
    }
    
    template(in A, out dest) store(A, word[8] location dest) {
        0x8D [<dest] [>dest]                        /* STA absolute */
    }
    
    routine(uses A, out score) main {
        load(A, 0)
        store(A, score)
    }

Note that this is enough information to consider the version of `main` show above
to be valid, and to produce an assembly-language program for it, while rejecting

    routine(uses A, out score) main {
        store(A, score)
    }

with a message such as `'store' expects 'A' to be meaningful but in its application in 'main' it is not`,
and also rejecting

    routine(uses A, out score) main {
        load(A, 0)
    }

with a message such as `'main' does not give a meaningful value to 'score'`.

### In more detail

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
definition used in a template application is retrieved based on the types of
the declarations given as parameters.  This is like method overloading in languages
such as Java.  Actually, it goes further, in that a template definition can
list a particular declaration, rather than a type, as one of its parameters.
This definition will be selected when this exact parameter is used.  The `load`
and `store` templates in the above example demonstrate this for the declaration `A`.

### Meaningful values

The most prominent property of declarations that PolyRical tracks is _meaningfulness_.
This is very similar to how many C compilers track _defineness_ of variables, and
are able to warn the user if the code uses a variable that is not defined, or may not
be defined in all cases.

Meaningfulness is controlled by three constraints on each routine type and on each
template:

*   `in` asserts that a declaration must be meaningful before the routine or template is applied
*   `out` asserts that a declaration will be meaningful after the routine or template is applied
*   `uses` asserts that a declaration will not be meaningful after the routine or template is applied

The meaningfulness of all other declarations is preserved when a routine or template
is applied.

### Control flow

Static analysis is trivial if all programs are entirely straight-line code; the
complications come up when branching occurs.

But branching is also an opportunity, because every time a branch occurs, the
analyzer has more information about what conditions must pertain in each branch.
(cf. flow typing)

For example, if we branch on the carry flag, we know that, in the code that we
branch to, the carry flag must always be set; and in the code where the branch
was failed to be taken, the carry flag must always be clear.

(TODO write more about how we'd like to capture control flow in templates but
we probably won't to start because that's very very hard.)
