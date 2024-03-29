# NAME

Reflections on Semantics of a Core Perl Object System

# VERSION

This is version 0.01 of this document.

# AUTHOR

chromatic

# CAVEAT!

This may be wrong. This is a work in progress. Language design is more
aesthetic than it is empirical. I reserve the right to change my mind.

This reflects my preferences for an object system, which may not be yours and
are certainly not everyone's.

# DESCRIPTION

What should an object system do and why and what should that _mean_?

# ESSENTIALS OF AN OBJECT SYSTEM

A language's object system should accomplish several goals. These goals often
overlap and sometimes conflict, so they are presented here in no particular
order of importance or priority.

## Allow Language Extension

An object system should allow language users to extend the capabilities of the
language in ways the language designers have not foreseen. This does not
necessarily imply _adding_ new language features by creating classes and
objects, but it does imply addressing unanticipated problems and activities.

For example, [XML::Rabbit](https://metacpan.org/pod/XML::Rabbit) uses Moose and
its metaclass system to provide an object-based approach to working with XML
documents so that users can interact with structured data through language
facilities based on classes, objects, and methods rather than diving deeply
into [semi-structured
text](http://www.modernperlbooks.com/mt/2010/10/structured-data-and-knowing-versus-guessing.html).

A good object system like Moose (and a strong language which supports
extension) allows things like this without anyone having guessed that something
like this would be useful and broadly usable.

## Encourage Correct Code

I strongly believe that language design should concentrate on making the right
thing to do easier and simpler and more obvious than the wrong thing to do.
(This is why [sum types are better than return
values](https://medium.com/fullstack-academy/better-js-cases-with-sum-types-92876e48fd9f)).

In terms of an object system, important questions here include:

 * is it possible to construct unusable objects?
 * do objects follow fundamental language features such as ordering,
   debuggability, and argument passing?
 * does class and metaclass syntax introduce edge cases to the rest of the
   language?
 * do object feature introduce correctness risks when combined with other
   features (here I'm thinking of things like pass-by-reference or
   pass-by-value semantics, serialization, visibility, synchronization and
   concurrency, for example)
 * how far does "correct" go when it comes to things like accessor/attribute
   access, the open/closed principle, and any interaction with any type system
   in effect

## Encourage Good Naming and Modeling

This is probably the fuzziest argument in these reflections, but there's
something that seems a little bit wrong to me in hybrid languages and I've
never been able to express it well.

When I asked "Why are some things primitives in Java and some things objects or
at least objecty?" I started thinking about traits and roles (and the
graybeards and/or smart alecks told me Smalltalk answered all my questions,
which is fine, but most of us aren't using Smalltalk).

When I asked "What's the difference between an identity enforced in the type
system allomorphically and a tag attached to a struct?" I didn't get much of an
answer because the problems where you want the second thing aren't often the
same problems where you want the first thing, but that's an interesting place
for language designers to think.

When I asked "why is there an [object relational psychological
mismatch](https://web.archive.org/web/20190224040722/http://wiki.c2.com/?ObjectRelationalPsychologicalMismatch)?"
I started to wonder if the _representation_ of data was an important enough
concept that it deserves its own category of object system design. After all,
the [6model
REPR](https://github.com/perl6/nqp/blob/master/docs/6model/repr-compose-protocol.markdown)
turns out to be a pretty good idea in both design and practice.

What does this have to do with naming?

Computers don't care about names, but humans do, and being able to put the
right name around a big bag of data and behavior means you can reason about
what's in and what's out and what's represented and what happens and why, and
you can discuss that with other people. That's essentially a modeling exercise.

If, for example, your language system requires you to extract some sort of
header file or named interface with API and constant and enumeration
declarations so that other people can use your code, you've introduced a layer
of complexity that ought to pay off by making compilation times shorter or
distributing code easier or consuming APIs simpler or reviewing documentation
possible or navigating and refactoring and debugging in an IDE possible.

In addition, the easier you make it for people to write code saying "I have a
primitive value that's always going to be an integer because that's what my
relational database thinks it is, but it should never be a zero or negative
value, never change, never be confused with a similar type of value that
represents the primary key of a different database table and should never
participate in any mathematical operation besides equality against the same
type or maybe hashing", the more you allow them to model concepts that might be
really valuable in specific domains.

Maybe instead they're thinking about distinguishing between two string types,
one of them which is HTML-safe and the other which isn't.

Maybe instead they're thinking about different representations of Unicode data.

Maybe they're thinking about something entirely different you haven't thought of.

Or maybe they're not really thinking about something you can put a name on at
all, and they just want something that conforms enough to an expected interface
they can accomplish something quick and dirty and inline, and an anonymous
class or subclass is good enough and that's fine too.

## Leave the Core Language Open to Extension

This doesn't mean that you _should_ let people monkeypatch `Array` to add
`#convertToBag` or `#removeDuplicatesInPlace` or `#newFromTuple` methods, but
you should discuss whether that's something you want.

This doesn't mean that you _should_ accept `HouseStyleString` everywhere you
allow a string operation, such as `concat` or `substr` or `print`, but you
should have that discussion too.

Here's the specific argument for Perl, though: Perl is a language which
encourages lexical modifications. For every
[UNIVERSAL::require](https://metacpan.org/pod/UNIVERSAL::require), there's an
argument that code elsewhere might do something similar where global action at
a distance means multiple things are going to conflict and something has to win
and the rules about what wins are clear but evaluating those rules means lots
of frustrated debugging time.

I _think_ this argument argues strongly for something like
[autobox](https://metacpan.org/pod/autobox) and, if that's true, a strong
role-based definition of core types. (If nothing else, that might force a
cleanup of the oddness in [overload](https://metacpan.org/pod/overload).)

Even if you don't go that far, you should attempt to reduce the distance
between "things that are built into the language" and "things that users add to
the language" if for no other reason that that's how user-defined functions are
supposed to work, so why not do the same work for user-defined methods and
user-defined classes?

I realize this sounds like a silly and trivial point, but it's not, especially
when it comes to meta-object capabilities like class declaration,
introspection, and reflection. Consider carefully what's written in Perl
currently in `UNIVERSAL.xs` versus `UNIVERSAL.pm` and ask whether a class
defined in pure Perl has access to the same capabilities as a class defined in
XS.

In both cases, you can squint and argue "they're both modifying the
capabilities of the language itself", but that's the old argument from
"everything is a Turing machine", and it's not an honest argument.

Perhaps the real argument is less "you shouldn't require people to write
`autobox` in XS to get it to exist" and more "`autobox` should be a language
feature", except the argument isn't primarily about `autobox` at all -- it's
about the kinds of things that are like `autobox` that someone will want in the
future.

## Implement Coercion in the Right Places

Consider again the example of modeling a database connection. If you use
[DBI](https://metacpan.org/pod/DBI), you'll end up passing in some sort of
connection information, including some or all of:

  * the `DBD` class of the connector (SQLite, PostgreSQL, CSV, etc)
  * the hostname of a database server
  * the port of a database server
  * the path to a file
  * the username of a database user
  * the password of a database user

Much of this data can and should be ephemeral; an object that persists
authentication and authorization credentials after making an authenticated
token is doing something risky.

It's easy to understand the idea of "a constructor needs some data to construct
an object, but should not expose that data after successful object
construction", but the subtle point is that this is the same _type_ of
operation seen elsewhere in larger and smaller places.

Consider also the example of constructing a
[DateTime](https://metacpan.org/pod/DateTime) object from epoch seconds.
`DateTime->from_epoch( epoch => $epoch_seconds )` is a constructor the same way as:

    DateTime->new(
        year      => 2003,
        month     => 10,
        day       => 26,
        hour      => 1,
        minute    => 30,
        second    => 0,
        time_zone => 'America/Chicago',
    );

... and `DateTime->today`. In all three cases, `DateTime` takes some (or none)
incoming data and converts it to a canonical internal representation.

What about coercing something that isn't a constructor? What about coercion in
an accessor method? Here's [Moose Subtypes and Coercion example
code](https://metacpan.org/pod/distribution/Moose/lib/Moose/Cookbook/Basics/HTTP_SubtypesAndCoercion.pod):

    coerce 'My::Types::URI'
    => from 'Object'
        => via { $_->isa('URI')
                 ? $_
                 : Params::Coerce::coerce( 'URI', $_ ); }
    => from 'Str'
        => via { URI->new( $_, 'http' ) };

    has 'uri'  => ( is => 'rw', isa => 'My::Types::URI', coerce => 1 );

This declares an attribute `uri` with a getter and a setter. When the setter
receives a string, it tries to create a new `URI` object.

In all three cases -- explicit `new` constructor, alternate constructor not
named `new`, single-attribute setter -- the object system provides a mechanism which:

  * validates the well-formedness of the incoming data (`Object` and `Str` and
    `My::Types::URI` are acceptable types for `uri`)
  * validates the correctness of the incoming data (if the DBD can't be found
    or the PostgreSQL password is wrong, you don't get a DBI handle)
  * transforms the incoming data into the modeled internal representation (a
    `DateTime` object has time and date and time zone information)

It's important to recognize all three features of this mechanism while also
recognizing that the _scope_ of the mechanism should be different based on the
nature of what you expect to get out of the coercion. In other words, I don't
care about the internal representation of database connection information after
I have a database handle, but I do care about the internal representation of
`uri` because that's what I'm going to get back if I ever call the accessor.

Without getting too deeply into syntactic concerns, the coercion must:

  * define obviously the interface to a constructor or attribute setter
  * represent the scope of data provided to the previous item appropriately
  * expose no more data than necessary

Moose is heavily attribute-centric, which means it tends to get this wrong when
it comes to constructor coercion and get it pretty right when it comes to
accessor coercion. Imagine if accessor coercion instead looked something like
this:

    has '_maybe_uri_string', is => Str,    required => unless('_maybe_uri_object');
    has '_maybe_uri_object', is => Object, required => unless('_maybe_uri_string');

    has 'uri'  => ( is => 'rw', isa => 'My::Types::URI', coerce => 1 );

    sub _coerce_uri_from_maybe {
        my $self = shift;

        if (my $uri_string = $self->_maybe_uri_string) {
            return $URI->new( $uri_string, 'http' );
        }

        if (my $uri_object = $self->_maybe_uri_object) {
            return $uri_object if $uri_object->isa('URI');
            return Params::Coerce::coerce( 'URI', $uri_object );
        }

        # there's a missing branch here
        # this is not ideal
        # do not use this code
    }

By forcing object instantiation into a single path of generated `new()` (though
with `BUILD()` and `BUILDARGS()`), Moose fails to account for coercion at the
object constructor level while doing a pretty good job of exposing its use at
the attribute level. Moose recognizes that there may be multiple valid ways of
setting an attribute value but makes it difficult to model something like the
`DateTime` approach of instantiating an object with explicit time and date
attributes or epoch seconds or the current time.

This could be solved with a Factory pattern, but that's working around a
deficiency in the object model. Strong support for class methods which
themselves wrap object construction may also work; this is worth considering in
more detail.

## Focus on the Core of an Object System

What are the most important features of an object system? If you look at what
Moose spends time and code and documentation on, you might think that's "reduce
the amount of code people have to write around accessors and constructors".
(Again, that's not a fair argument applied to the aims of Moose, and Moose is
great, but Moose goes where it goes because that's where Moose _can_ go.)

I think the list instead looks something like this:

 * allow users to define (and optionally name) their own first-class entities
 * ... which contain data
 * ... which allow named operations on that data
 * ... which respect [allomorphic equivalences](http://modernperlbooks.com/mt/2009/04/the-why-of-perl-roles.html)
   and
   [substitutions](https://www.stephanboyer.com/post/132/what-are-covariance-and-contravariance)
 * ... which participate effectively and correctly with existing language
   features

I keep mentioning "allomorphism" and implying "substitution" and I don't want
to get into type theory (partly because I always have to look up the difference
between "covariant" and "contravariant") because it's an important distinction
Moose already respects but also because I believe it's _inherently Perlish_.
That's why `overload` exists at all, after all. Throwing any scalar into double
quotes and expecting interpolation without crashing your program is
quintessentially Perlish, even if Perl isn't exposing its idea that "you're
putting something which `does Stringifiable` here".

If this is a feature inherent to Perl -- if this allomorphism is just me naming
something that Perl stole from the Bourne Shell and other places -- then an
object system should participate in it fully the same way you expect the
package name associated with a reference by `bless` not to get lost when you
pass that reference to a function or put that reference in an aggregate data
structure.

Put another way, the semantics of user-defined entities should match the
semantic capabilities of built-in entities and user-defined entities should
have access to the semantic capabilities of the core language. That's what
"first-class" support means, right?

## Take Sides on Mechanism But Don't Take Too Many Sides

My larger programs try to focus on composition and allomorphism over
inheritance, but I'll happily use inheritance where it makes things cleaner and
clearer.

A good object system will do its best to encourage users to pick the right
approach for their problem domain. I think this means first-class roles support
and a good metaclass system where user-defined metaclasses are as easy to use
as the built-in metaclass, but I think it's too late for Perl to jump to
something like prototypes over classes, for example.

Should a core object system encourage people to define roles? I believe so.

Should a core object system make inheritance more difficult to use than
composition or delegation? I'm not sure. I'd rather it make composition and
delegation easier to use than inheritance.

Should a core object system make private attributes the default? I believe so.

Should a core object system make public accessors for all attributes? I believe
not. Should it forbid users from making public accessors? Never.

Should a core object system build objects around blessed hash references? I
believe not -- but if it does, it should offer me the option to select
different representations where a single scalar (my primary key of a database
table), a generator/producer (a list of items read from disk or the network or
user input), a backing store (an XML document in the case of `XML::Rabbit` or
the hypothetical `JSON::Rabbit` which ought to exist if it doesn't already), or
something else.

I believe these opinions should be examined and discussed and adopted after
discussion, not copied from extant systems without consideration. Thus the rest
of this document....

# SUGGESTED SEMANTICS OF A NEW OBJECT SYSTEM

## Different Types of Coercion are Distinct

A good object system will recognize that there may be multiple valid ways to
construct a coherent object.

A good object system will recognize that constructor parameters should not
necessarily be treated the same way as attributes.

A good object system will not force object construction to go through a single
parameter validation, coercion, and population mechanism.

## Attributes are Private to Classes, Roles, and Subclasses

## Methods are Not Functions

## Classes Imply Roles Imply Allomorphic Types

## Attribute Representations are Opaque

## Attribute Representations are Substitutable

See "Different Types of Coercion are Distinct" for a variant of this position.
A SQLite database handle may operate on an in-memory database just as well as a
file-backed database. (In Unix terms, the file-backed database may even use a
RAM disk, which suggests that representation independence is a deeper principle
than post-Smalltalk object systems.)
