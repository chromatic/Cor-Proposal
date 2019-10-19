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

### Moose Implementation

### Moose Syntax
