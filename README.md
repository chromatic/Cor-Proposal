# NAME

Reflections on Semantics, Syntax, and Implementation of a Core Perl Object System

# VERSION

This is version 0.01 of this document.

# AUTHOR

chromatic

# CAVEAT!

This may be wrong. This is a work in progress. Language design is more
aesthetic than it is empirical. I reserve the right to change my mind.

# DESCRIPTION

The [Cor minimal OO system
proposal](https://gist.github.com/Ovid/68b33259cb81c01f9a51612c7a294ede) offers
several possibilities for an enhanced core Perl object system. It relies on the
hard-won experience of the Moose ecosystem as well as the extant Perl object
system.

Before any such proposal can be adopted, it should consider several things:

 * [Semantics](semantics.md)
 * [Implementation](implementation.md)
 * [Syntax](syntax.md)

# BACKGROUND

Before _those_ can be considered, the _constraints_ imposed by the language,
its implementation, history, interoperability, future plans, and competing
designs should be considered.

First, some disclaimers.

## Disclaimers

"Moose" in these reflections means [Moose](https://metacpan.org/pod/Moose),
[Moo](https://metacpan.org/pod/Moo), et all (and maybe even
[Moops](https://metacpan.org/pod/Moops), where its design and implementation
does not conflict with the broader point.)

Programming language design is a matter of taste.

Programming language features such as enforceability of type systems,
meta-syntactic constructs, extensibility, data visibility and encapsulation,
performance costs of these features, and more are a matter of taste, though not
solely a matter of taste.

Constraints make design interesting but competing (and poorly-communicated or
-understood) constraints make communication difficult.

Everyone is generally doing the best they can.

## Why is Moose Great 'Til It's Gotta Be Great?

Moose is the best object system Perl has ever had, especially when you can use
something like the Moops flavor of Moose which improves syntax, reduces
boilerplate, and gives you the option to improve semantics.

Moose has its flaws, though -- not because it's poorly designed or implemented,
but because it exists in a system that doesn't allow it to be great. For
example, the [How Moose made me a bad OO
Programmer](https://www.youtube.com/watch?v=783CBT1r1DU) talk by Tadeusz
Sośnierz describes some of the drawbacks of Moose as currently implemented.

This is not a condemnation of Moose qua Moose (Moose is great!). This is a
recognition that Moose or any other Perl object system won't be great as long
as the constraints keeping Moose from being great are in place.

_If_ this is true (and I hope to convince you that it is), then any new object
system should attempt to remove, obviate, or address these constraints of
semantics, implementation, and syntax so that it can become greater than Moose.

### Moose Semantics

This section describes semantics of Moose which are implicit in its use and
implementation and the principles of which may be induced from real-world Moose
experience. In other words, I may be wrong about the principles and you may
disagree about semantics. Even if that is the case, this discussion is worth
having _because people get this wrong when using Moose in real projects_.

#### Moose is Attribute-Centric

Tadeusz explains this very well in his video. You can derive this fact about
Moose by observing that the most complex syntactic elements Moose invented
exist to manage attribute declaration, initialization, coercion, et cetera.

When you create a Moose class, you get:

 * a constructor which sets attributes
 * attributes accessors
 * a "hidden" attribute storage mechanism
 * optional coercions
 * optional initialization mechanisms for attributes (laziness, default values)
 * metaclasses

Set aside metaclasses, superclass relationships, roles, and method modifiers
for a moment. If you had no benefit out of Moose other than managing attributes
and accessors and a constructor, would you say that Moose is substantially
better than the extant Perl OO system? Probably yes!

Yet this attribute-centric model makes a few important semantic assumptions
which are central to Moose usage:

 * attributes _generally_ have accessors
 * there should _generally_ be a single constructor
 * a hash-based attribute storage mechanism is _generally_ the right thing
 * attribute accessors generally mean setters, clearers, and accessors
 * attributes can be modified by accessors after object construction

Here the word _generally_ means "most of the time" or "in common practice". In
other words, by not actively discouraging this behavior, Moose implicitly
recommends it. This is not necessarily the wrong decision and it is a
defensible decision given the constraints in which Moose must exist, but it
should not be an unquestioned decision.

#### Moose Offers a Weak View of Object Construction and Initialization

Tadeusz highlighted one extremely important point, namely that *constructor
parameters _generally_ map to attributes*.

Because of Moose's attribute-centric design, constructor arguments are
_generally_ expressed as attributes even if they may be irrelevant once an
object is correctly constructed. For example:

    use Moops;
    Modern::Perl;

    class SomeProject::DB :ro {
        use DBI;

        has 'connection_info', required => 1;

        has 'dbh', lazy_build => 1;

        method _build_dbh {
            my $connection_info  = $self->connection_info;
            my $connection_attrs = $connection_info[-1] ||= {};

            $connection_attrs->{RaiseError} = 1;

            return DBI->connect( @$connection_info );
        }
    }

Though this code has some drawbacks, it is a reasonable implementation of a
database wrapper which allows parametric connection information and lazily
connects to a database as needed. These are reasonable semantics to support on
their own, but they demonstrate some drawbacks of Moose's design:

 * `connection_info` persists through the life of the object, which isn't great
   because it may contain database connection information, such as a password
 * there's no validation of the _shape_ of data within `connection_info`
   outside of the lazy builder for `dbh`, which means that an object can be
   constructed successfully but the database connection will fail because of
   poor data (and not only because the database has gone away, the password has
   expired, the network is losing packets, etc)
 * Moose's lazy attributes are expressed using the same mechanism as other
   attributes, so they have setters and getters and clearers even when this may
   not be appropriate

The *Semantics* discussion will discuss these issues in more detail.

On the positive side, Moose's semantics do support dependency injection for the
`dbh` attribute, such that that attribute can be passed in to the default
constructor to bypass these issues altogether (and to use another database
connection class if the hard-coded `DBI` is not appropriate in this case).
However, this raises the issue that `connection_info` is still required and
introduces the interesting tension between pushing validation of the shape and
correctness of `connection_info` into the attribute itself (such that it can be
validated in the constructor) or leaving it as is, in which case an object can
be constructed with dependency injection in the default constructor with code
like:

    my $dbh = SomeProject::DBI::Pool->new( ... )
    my $db  = SomeProject::DB->new({ dbh => $dbh, connnection_attrs => [qw( unused ignore foo bar )] });

That means writing do-nothing code to avoid having the wrong behavior by default.

#### Moose Favors the Wrong Things

Again, this is not a criticism of Moose qua Moose but instead an
acknowledgement that Moose's design is optimized for certain goals that are not
necessarily the correct goals for a stronger core object model.

For example, *Moose classes are not immutable by default*. This produces a
speed penalty that can be obviated by adding the boilerplate to the end of
every class declaration:

    __PACKAGE__->meta->make_immutable;

This is similar in spirit to the boilerplate that every Perl package must end
with a true value, one which `Moops` at least attempts to clear up with its
`class` and `role` keywords.

*Moose classes favor inheritance over composition*, especially with regard to
its method modifiers. While the provenance of these modifiers has a
distinguished language heritage, the putative source of their design in Common
Lisp demonstrates the difference between a class-based object-dispatched method
system and a functional multiple dispatch system.

In this case, I can't decide how serious I am with this point, because I'm not
sure I understand if these method modifiers would be necessary as implemented
if Moose favored other composition or aggregation or delegation semantics over
inheritance; would `around` or `before` be necessary?

*Moose uses the word "parameterized"* instead of "parametric" to describe
partially-closed-while-unapplied roles, which is objectively a violation of
sensible linguistic taste.

*Moose makes lightweight classes difficult* in the sense that this code should
be easy to make more correct with better types:

    method update_needle_description( Int $haystack_id, Int $needle_id, Str $description ) {
        $self->update_needle_description_sth->execute( $description, $needle_id, $haystack_id );
    }

Using the same type of `Int` to represent two different types of foreign keys
means that this method has a needle-haystack problem. (In fact, this is a bug I
fixed in code I wrote this week, so it's a tender subject.) If I had a specific
integer type for a Version ID versus an Ingredients ID, I could rely on the
[Kavorka](https://metacpan.org/pod/Kavorka) signature/type system to tell me
that I'd written the consumer of this method incorrectly.

Why would I want a class to represent these integers rather than a
specifically-typed integer? That's a good question. First, because Perl lacks a
mechanism of identifying subtypes of primitives without attaching some sort of
package identity (or doing deeper, more esoteric magic). Second, because I know
I'm unlikely to lose that object identity by passing objects around because I'm
not doing anything to lose the blessedness of a reference.

I _could_ have gone through the work of defining classes to represent the
primary keys for every table in this application, but I didn't in part because
I didn't think I'd need it because the overhead of doing so felt so silly. (I
grow more and more sympathetic to this argument the more I think about
[Primitive
Obsession](https://www.jamesshore.com/Blog/PrimitiveObsession.html)).

##### Undercutting the Needle/Haystack Class Argument

One could easily say "This is a good example of why you should use a real ORM"
which hides these details, and that's not a bad argument on its face, but that
only applies to primitives used for primary keys in relational systems and
ignores all of the other primitives which could benefit from encapsulation,
immutability, verifiably-correct-at-construction-time features.

One could also say "This demonstrates that SQL is verbose and leads to
hard-to-use APIs when you hard-code the order of bound parameters in a query",
in which case the answer is that correctness and performance are not always
enemies but they're not always the best of friends.

### Moose Implementation

What do you get when you look inside a Moose?

#### Moose is Built on Existing Perl Primitives

One of the good things about Moose is that it interoperates pretty well with
existing Perl core-OO code. This is because Perl's core OO system provides
mostly sufficient primitives, except where it doesn't. Moose is built on:

 * `@ISA`
 * `UNIVERSAL`
 * method dispatch
 * blessed hashes
 * packages
 * method invocation is function invocation

"How should a new core object system behave with existing code" is a semantics
and implementation question. It comes up here only as it influences how Moose
works and how Moose is used.

#### Moose is Export-Centric

Why does Moose export `strict` and `warnings`? That's a policy decision! I tend
to agree with the policy in general, but as a mechanism of an OO system it
seems questionable, unless you realize that one underlying goal of Moose is to
reduce boilerplate code.

Again, this is a good goal in and of itself, but it shows that Moose blurs the
lines between the explicit semantics of an object system and the behavior of a
program as a whole.

At least these pragmas are lexically scoped. Yet what does this do?

    package SomeClass;

    use Moose;

    has 'attribute', is => 'rw';

    package OtherClass;

    $foo = bar();
    has 'another_attribute';

That's right; `has()` is out of scope outside of the `SomeClass` package, but
`strict` is in scope.

This is mostly not Moose's fault, except that it brings the effect of lexical
pragmas with package-scoped exports.

#### Moose is Function Export-Centric

Maybe that should be _closure_ export-centric. For Moose to do its magic, it
has to carry around specific knowledge of a context which is created when you
`use Moose` or `use Moose::Role`. That means what Moose exports is a closure.
Sometimes.

This is important because, for metaclasses and
partial-classes-under-construction and you-didn't-make-it-immutable-yet-or-ever
classes to work, something somewhere has the baggage duty of keeping track of
data and metadata because the Perl core sure isn't doing it.

Sometimes you see the seams in this approach:

    package SomeClass;

    use Moose;

    has 'attribute', is => 'rw';

    package OtherClass;

    SomeClass::has( 'another_attribute' );

That obviously _shouldn't_ work, but what does the error message suggest is going on?

What happens if you accidentally call one of Moose's exported functions in a
subclass? (Yes, that's what something like
[namespace::autoclean](https://metacpan.org/pod/namespace::autoclean) is
for.)

Moose's implementation relies on a combination of:

 * a consumer using a Moose entrypoint module
 * exporting functions/closures into a namespace
 * exporting pragmas into a lexical scope

This is a natural, normal, and expected way to implement Moose in a
backwards-compatible way which requires no additional core OO support. However
it is not necessarily the best way to implement a new core object system and
its design decisions should not remain unchallenged when fundamental
constraints that were in place for Moose are removed.

### Moose Syntax

Moose offers some syntax. Does it offer enough syntax? No. Does it offer the
right syntax? Given what it's trying to accomplish and the environment it found
itself, mostly. Yet this syntax should not go unchallenged.

#### Moose Infers Classes

What's a Moose class? More importantly, what does it look like when you're
reading code?

A Moose class is a package which doesn't define its own constructor and which
uses the module `Moose`. This allows Moose to export its functions into a
namespace (but not _lexically_).

Any package into which Moose imports is now a Moose class, except when it
isn't:

    use Modern::Perl;
    use Path::Class;
    use File::Find;

    package Foo::Bar    { use Moose };
    package Path::Class { use Moose };
    package File::Find  { use Moose };

    package main;

    sub main {
        my $fb = Foo::Bar->new;
        my $pc = Path::Class->new; # no constructor here
        my $ff = File::Find->new;  # no constructor here

        return 0;
    }

    exit main( @ARGV );

Then again, sometimes it is:

    use Modern::Perl;
    use Path::Class;
    use File::Find;

    package Foo::Bar    { use Moose };
    package Path::Class { use Moose; __PACKAGE__->meta->make_immutable; };
    package File::Find  { use Moose; __PACKAGE__->meta->make_immutable; };

    package main;

    sub main {
        my $fb = Foo::Bar->new;
        my $pc = Path::Class->new;
        my $ff = File::Find->new;

        return 0;
    }

    exit main( @ARGV );

While it's obviously silly to take modules which weren't designed as classes
and try to force them into Moose behavior, it's odd that both of these things
are true:

 * importing Moose into a package from outside doesn't Moosify it
 * ... unless you add an optional optimization that is supposed to be
   semantically neutral

#### Moose Doesn't Infer Roles

What's a Moose role? More importantly, what does it look like when you're reading code?

A Moose role is a package which doesn't define its own constructor and which
uses the module [Moose::Role](https://metacpan.org/pod/Moose::Role).

This is a very minor point, but when the obvious visual difference is "which
function-exporting module did I import", that's a small syntactic marker for a
large semantic difference. Moops does this better.

#### Insert Punctuation (to Clarify (Sometimes?!))

I've never liked this attribute declaration syntax:

    package Foo;
    use Moose;

    has 'message' => (
        is      => 'rw',
        isa     => 'Str',
        default => 'Hello, I am a Foo'
    );

This is parsed and executed equivalently to:

    has( 'message', 'is', 'rw', 'isa', 'Str', 'default', 'Hello, I am a Foo' );

... though the use of the `=>` quoting arrow does make it clear that there is a
syntactic and semantic pairwise association between its lvalues and rvalues.

While one could argue successfully that grouping the attributes of the
attribute (the meta-attributes?) in parentheses serves a visual distinction in
the same way, quoting the attribute name is completely unnecessary
syntactically with the pairwise quoting arrow.

As petty as "why are there parentheses here and unnecessary quotes are
unnecessary" is (and it's very petty), it's even more petty to note how much
visual space attribute declaration takes up within a Moose class. I personally
believe this is because:

 * writing good accessors is a lot of work (so Moose reducing the boilerplate
   is generally a good thing!)
 * writing good classes and objects means limiting access to nouns and
   concentrating on verbs (which is a semantic argument where Moose *and* Perl
   may not want to take the same strong stance I do)
 * what does an attribute _mean_ anyhow?
    ** Is it derived data?
    ** Does mutating it change the behavior of the object?
    ** Is this class just a struct with behavior? Should it be? Should that be _impossible_?

Note that the syntax for various attribute behaviors is clunky (I _think_ this
is because of the desire to make all of this syntax look and act superficially
consistently, but I could be wrong about this). For example, delegation (here,
`handles`) uses a hash reference:

    package User;

    has 'last_login' => (
        is      => 'rw',
        isa     => 'DateTime',
        handles => { 'date_of_last_login' => 'date' },
    );

Default attribute values (here, `default`) may be specified inline as literals
or function references (in which case, presumably you can't use a function
reference as a default value unless you wrap it in a function reference
itself):

    has 'size' => (
        is        => 'ro',
        default   => sub { ( 'small', 'medium', 'large' )[ int( rand 3 ) ] },
        predicate => 'has_size',
    );

    has 'unworking_reference' => (
        is        => 'ro',
        default   => [], # don't do this
    );

    has 'gift_wrapped_function_reference' => (
        is      => 'ro',
        default => sub { sub { 'here is my callback, finally' } },
    );

A builder is called by name (so it can be overridden), but a trigger is
provided as a subroutine reference which is called as a method (which makes it
more difficult to override in a subclass).

[This is a lot to keep in mind](https://metacpan.org/pod/distribution/Moose/lib/Moose/Manual/Attributes.pod).

#### Moose is Method-Centric

How do you access attributes? Moose blesses hashes, so... you use accessors.
(Encapsulation is a good thing, but it implies some encapsulation access
mechanism and that's methods in Moose. That's why getters and setters and
clearers and builders are so important in Moose's attribute-centric model.)

How can Moose be both method- _and_ attribute-centric? How does Moose handle
class methods? About the same way the Perl core does! Again, that's not Moose
making a bad decision; it's Moose being consistent with the design of the Perl
core.
