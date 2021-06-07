PolyRical
=========

PolyRical is a successor language to [SixtyPical][].

Like SixtyPical, it is a very low-level language that nonetheless supports advanced
forms of static analysis.  It aims to be based on a cleaner, more general theory of
operation than SixtyPical, and thus (ideally speaking) able to target architectures
other than the 6502.

[SixtyPical]: https://catseye.tc/node/SixtyPical

### A self-contained example

A PolyRical program consists of pragmas, templates, and declarations.  Typically one
would import a set of templates from a library using a pragma, but to convey the
flavour of the the language, here is a self-contained example program:

    word[8] register A
    word[8] location score @ 0xc000
    
    template(out A) load(A, word[8] value val) {
        "LDA #${val}"
    }
    
    template(in A, out dest) store(A, word[8] location dest) {
        "STA ${dest}"
    }

    routine(uses A, out score) main {
        load(A, 0)
        store(A, score)
    }

Note that this is enough information to consider the version of `main` show above
to be valid, and to produce (most of) an assembly-language program for it, while
rejecting

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
