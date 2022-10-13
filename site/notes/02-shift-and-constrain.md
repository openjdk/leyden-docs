
Selectively Shifting and Constraining Computation
=================================================

#### Mark Reinhold {.author}
#### 2022/10/13 {.date}


> The [goal][cfd] of Project Leyden is to improve the startup time, time
> to peak performance, and footprint of Java programs.  In this note we
> propose to work toward that goal by extending the Java programming
> model with features for _selectively shifting and constraining
> computation_ by means of _condensing code_.  We also propose an initial
> research and development roadmap.


We can often improve a program’s startup time, warmup time, and footprint
by _shifting_ some of its computation temporally, either forward to a
point later in run time (e.g., via lazy initialization) or backward to a
point earlier than run time (e.g., via ahead-of-time compilation).  We
can further improve performance by _constraining_ some of the computation
related to Java’s dynamic features (e.g., class loading, class
redefinition, and reflection), which enables better code analysis and
thus even more optimization.

In Project Leyden we will implement these shifting, constraining, and
optimizing transformations as _condensers_.  A condenser is a program
transformer that runs in a phase between compile time and run time.  It
transforms a program into a new, faster, and potentially smaller program
while preserving the meaning given to the original program by the Java
Platform Specification.  We will evolve the Specification as required to
admit and support these kinds of transformations.  We will also
investigate new language features that allow developers to shift
computation themselves, thereby enabling further condensation.

The condensation model gives developers great flexibility: They choose
which condensers to apply, and by doing so they choose whether and how to
accept constraints that limit Java’s natural dynamism.  There’s no need
to tolerate a one-size-fits-all model of startup, warmup, and footprint
optimization.

The condensation model also gives Java implementations considerable
freedom: As long as a condenser preserves program meaning and does not
impose constraints other than those accepted by the developer, an
implementation has wide latitude to optimize the result.

The performance characteristics of a condensed program are an emergent
property of the condensers applied to the program.  If a developer
chooses condensers that shift sufficient computation from run time to
earlier phases -- which may require accepting many constraints -- then a
conforming Java implementation could even produce a fully-static
platform-specific executable.

[cfd]: https://mail.openjdk.org/pipermail/discuss/2020-April/005429.html


Shifting computation
--------------------

The best way to improve startup time, warmup time, and footprint is to
identify computation that we can simply eliminate.  Failing that, we can
_shift_ computation temporally, forward or backward in time.  We can
shift it forward to a point later in run time, in the hope that it won’t
always be necessary.  We can shift it backward to a point earlier than
run time, such as compile time.  We can shift computation that’s
expressed directly by a program (e.g., execute a `for` loop), and we can
shift computation that’s performed by the runtime system indirectly, on
behalf of a program (e.g., compile a method to native code).

The concept of shifting computation in time is not new; indeed, Java
implementations already have many features that can shift computation.
Some of these features work automatically; for example,

  - Compile-time constant folding shifts direct computation, namely the
    evaluation of simple expressions, backward in time from run time to
    compile time, and

  - Garbage collection shifts indirect computation, namely the
    reclamation of memory, forward to a point later in run time.

Other computation-shifting features are optional, and must be requested
and in some cases configured; for example,

  - Ahead-of-time compilation shifts indirect computation, namely the
    compilation of Java code to native code, backward in time from run
    time to compile time, and

  - [Class-data sharing (CDS)][cds] shifts indirect computation, namely
    the parsing and verification of some class files and the
    initialization of some run-time data structures, backward in time
    from run time to archive-generation time.

Yet other computation-shifting features are inherent in the Java
programming language, and can be used by developers themselves to shift
computation.  Lazy class loading and initialization, for example, allows
developers to shift direct computation forward in time via techniques
such as the [initialization-on-demand holder idiom].

If a Java implementation shifts computation temporally then it must do so
within the bounds of the Java Platform Specification, in particular the
[Java Language Specification] and the [Java Virtual Machine
Specification].  These specifications do not allow computation to be
shifted in time arbitrarily.  Thus the shifting features available in
conforming Java implementations -- including all of those mentioned above
-- have been designed carefully to operate within the bounds of these
specifications, so as to ensure compatibility.

In Project Leyden we’ll explore additional techniques for shifting
computation, and where necessary we’ll evolve the Platform Specification
to accommodate them so that shifting computation preserves a program’s
meaning.  Some of these techniques might not require any specification
changes, e.g., expanding dynamic-proxy call sites into ordinary bytecode
prior to run time.  Others will definitely require specification changes,
e.g., resolving classes prior to run time.  Yet others will take the form
of new platform features that allow developers to express temporal
shifting directly in source code, e.g., [lazy static final fields][lazy].

[cds]: https://docs.oracle.com/en/java/javase/19/docs/specs/man/java.html#application-class-data-sharing
[Java Language Specification]: https://docs.oracle.com/javase/specs/jls/se19/html/index.html
[Java Virtual Machine Specification]: https://docs.oracle.com/javase/specs/jvms/se19/html/index.html
[initialization-on-demand holder idiom]: https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom
[lazy]: https://openjdk.org/jeps/8209964


Constraining Java’s natural dynamism
------------------------------------

Shifting computation often requires analyzing code: We must determine
exactly what computation to shift, and we must ensure that the shift will
preserve the program’s meaning per the Platform Specification.

In the case of Java, the platform’s dynamic features make code analysis
more difficult than for a conventional language: A running program can
load classes, redefine classes, and reflectively access fields and invoke
methods in ways that are impossible to predict.

One way to address this problem is to adopt the _closed-world
constraint_, which requires that all code executed at run time be known
and available for analysis at build time.  Arbitrary dynamic class
loading and reflection are, thus, forbidden.  As we know, however, not
all applications are well suited to this constraint since many libraries
and frameworks rely heavily upon class loading and reflection.  Not all
developers are, moreover, willing to live with the closed-world
constraint since tools that leverage it typically require developers to
construct and maintain fragile configuration files, and to endure
unusually long build times.

That’s why, in Project Leyden, we’ll explore weaker constraints as well.
We’ll look at ways to selectively and explicitly limit the dynamic
functionality of the platform in order to enable more and better shifting
of computation.  Developers should be able to choose, themselves, how to
trade functionality for performance according to the needs of their
applications.

For example, suppose we revise the specification to allow developers to
opt-in to the weaker constraint that some classes that they identify
cannot be redefined.  (In other words, any attempt to redefine such a
class would cause an exception to be thrown.)  Under this constraint a
conforming Java implementation could load, verify, prepare, and resolve
such classes prior to run time, storing the resulting metadata in the CDS
archive.  Shifting these computations earlier in time would enable
further optimizations, such as the elimination of some of the validity
checks required by CDS and ahead-of-time-compiled native code.  All of
this should result in a measurable improvement in startup time.


Phases of computation
---------------------

If we shift run-time computation forward in time then it still runs at
run time, during the normal lifetime of a JVM invocation -- if it runs at
all.  If we shift computation backward in time, however, prior to the
JVM’s invocation, then it must run in a different _phase_ of computation,
prior to run time and almost certainly in a different environment than
run time.

Originally Java had just two phases of computation: compile time (i.e.,
when you run `javac`) and run time.  In JDK&nbsp;9 we introduced _[link
time]_, a third, optional phase between the two in which a set of modules
can be assembled and optimized into a custom run-time image via the
`jlink` tool.  In JDK&nbsp;10 we implemented [application class-data
sharing][cds] which introduced a fourth optional phase, following either
compile time or link time, for the generation of the CDS archive.

We can, to varying degrees, shift computation from run time to any of the
prior three phases.

  - Compile time is a suitable phase for the localized shifting of
    computation backward in time (e.g., expanding dynamic proxies).  It’s
    not, however, a suitable phase for arbitrary time-shifted
    computations.  The Java language compiler is not guaranteed to have
    access to the entire program, and thus cannot always run such
    computations.  The compiler is, moreover, a transformer of source
    files into class files, and thus not in a position to shift
    computation in code that it’s not compiling (e.g., code in the JDK
    itself).

  - The CDS archive-generation phase can be suitable for some
    time-shifted computations, including computations of the JDK itself
    since archive generation requires the JDK to be available.  Archive
    generation does not, however, require access to the entire program,
    so like compile time it’s not a suitable phase for arbitrary
    time-shifted computations.

  - Link time, by contrast, is a suitable phase for arbitrary
    time-shifted computations.  The `jlink` tool must have access to both
    the JDK itself and the entire program in order to create a custom
    run-time image, thus it can always run such computations.

Why, however, should we limit ourselves to shifting computation backward
from run time to just the three phases of compile time,
archive-generation time, and link time?  More phases could be useful,
since different shifts of computation may best be applied at different
times.

Consider, for example, an application whose configuration is stored in an
XML file but does not otherwise require the JDK’s XML parser.
Schematically:

    class Configuration {
        private static final XML conf = XML.parse(CONFIG_FILE);
        private static final String serverName = conf.xpath("/conf/server");
        static String serverName() { return serverName; }
    }

    public class AppMain {
        public static void main(String ... args) {
            Server.start(Configuration.serverName(), ...);
        }
    }

We’d like to initialize the `Configuration` class in some phase prior to
run time, saving some time by storing the value of the `serverName` field
for use in later phases and then saving some space by removing the XML
parser, which is no longer required.

Initializing the `serverName` field, however, requires reading the
application’s XML configuration file, and in practice that file might not
be available to any of the existing three phases.  That could be the case
if, e.g., the application is linked into a custom run-time image for
deployment to multiple distinct server environments, each of which has
its own distinct XML configuration file.  By the time the image is
deployed it’s already been linked, and it’s too late to apply this
transformation.

So let’s generalize, and allow an arbitrary number of phases in which
time-shifting transformations and related optimizations can be applied.

[link time]: https://openjdk.org/jeps/261#Phases


Condensing code
---------------

The process of building a complete, runnable program is a sequence of
operations that transform code artifacts.  Compilation, e.g., transforms
a set of source files into a set of class files.  The artifacts involved
can take many other forms including, for example,

  - A JAR file containing an application’s code,

  - A set of JAR files, or a so-called [“fat” or “uber” JAR
    file][fatjar], containing both an application’s code and all of its
    dependencies,

  - A custom JDK run-time image containing an application’s code, its
    dependencies, and whatever modules of the JDK it requires, including
    a JVM, or even

  - A fully-static platform-specific executable containing application
    code, dependency code, and JDK code in native form only, without a
    JVM.

A _condenser_ is an optional transformation phase in which we:

  1. Perform some of the computation expressed in a code artifact,
     thereby shifting that computation from some later phase to the
     current phase.

  2. Apply optimizations enabled by that shift to transform the artifact
     into a new, faster, and potentially smaller artifact.  The resulting
     artifact can contain new code (e.g., ahead-of-time compiled
     methods), new data (e.g., serialized heap objects), and new metadata
     (e.g., pre-loaded classes).  It can also omit components of the
     original artifact that are no longer required (e.g., the XML
     parser).

  3. Possibly impose new constraints upon later phases (e.g., certain
     classes cannot be redefined).

Condensation has three critical properties.

  - Condensation is _meaning preserving_.  The resulting artifact runs
    the application according to the Java Platform Specification, just as
    the original artifact did.

  - Condensation is _composable_.  The artifact output by one condenser
    can be the input to another, so performance improvements accumulate
    across a chain of condensers.  By induction, a chain of condensers is
    also meaning-preserving.

  - Condensation is _selectable_.  Developers choose how to condense, and
    what to condense, and when to condense.

These three properties, taken together, give developers great flexibility
when building programs.  If you’re just testing or debugging, e.g., then
you can skip the condensers that you’d normally use for production builds
yet be assured that the meaning of your program will not change.  More
importantly, to the degree that shifting computations in time requires
accepting new constraints, you can choose how to trade functionality for
performance by choosing the condensers appropriate to your application.
There’s no need to tolerate a one-size-fits-all model of startup, warmup,
and footprint optimization.

Returning to the example above, we could optimize our linked run-time
image for each server environment as part of the deployment process.  We
could apply, at deployment time, a condenser that initializes selected
classes at build time (e.g., `Configuration`), followed by a condenser
that removes unused components (e.g., the XML parser).

[fatjar]: https://dzone.com/articles/the-skinny-on-fat-thin-hollow-and-uber


Specifying condensers
---------------------

Condensers run program code, transform program code, and sometimes place
constraints on what program code can do in later phases.  To ensure that
condensers run code correctly, and transform and constrain code only in
ways that preserve meaning, we must add the concept of condensers to the
Java Platform Specification.  We must also add any new opt-in constraints
(e.g., limited class redefinition) that we wish to leverage in
condensers.  Finally, we must extend the [Java Compatibility Kit
(JCK)][jck] to test the condensers available in a Java implementation.

We need not, however, extend the Specification to specify particular
condensers, nor extend the JCK to test precisely what any particular
condenser does.  The Specification defines the meaning of Java programs,
i.e., what the results of computations expressed directly by programs
must be, and that is what the JCK verifies.  The Specification
intentionally does not say how a program’s meaning can or must be
implemented.  A Java implementation is thus free to optimize within the
bounds of the Specification.

[jck]: https://openjdk.org/groups/conformance/JckAccess/


Performance is an emergent property
-----------------------------------

That the Platform Specification will not specify particular condensers
gives Java implementations considerable freedom.  As long as a condenser
preserves program meaning and does not impose constraints other than
those accepted by the developer, it has wide latitude to optimize the
result.

The performance characteristics of a complete, runnable program are,
therefore, not the direct consequence of any words in the Specification;
they are, rather, an emergent property of the condensers applied to the
program.  If a developer chooses condensers that shift sufficient
computation from run time to earlier phases -- which may require
accepting many constraints -- then a conforming Java implementation could
even produce a fully-static platform-specific executable.

Whether in Project Leyden itself we go so far as to implement
fully-static executables remains to be seen.  We can, however, enable
other Java implementations (e.g., some future version of the [GraalVM
native-image tool][ni]) to do so.

[ni]: https://www.graalvm.org/dev/reference-manual/native-image/


Roadmap
-------

We have two categories of work before us: Specify and implement the
concept of condensers, and research and develop specific condensers and
related new language features.

### Introduce condensers

As mentioned above, we must extend the Java Platform Specification with
the concept of condensers, and extend the JCK to test condensers.

In terms of implementation we’ll have to extend the JDK’s tools (e.g.,
`jlink`) to support condensers, and extend the formats of various code
artifacts (e.g., JAR files and run-time images) to accommodate new code,
data, and metadata.  Along the way we may discover that it’s best to
create new tools and, perhaps, a new, condensable artifact format.

### Research and develop new condensers and new language features

There are many possibilities; we list a few of them here.  Some of these
ideas are already being explored and prototyped by Project Leyden
contributors.  Work on most of them can proceed in parallel, and those
that prove fruitful can be delivered into main-line JDK releases
incrementally.

- Condensers that (likely) require no specification changes because they
  require no new opt-in constraints:

  - Expand dynamic-proxy call sites into ordinary bytecode

  - Expand internal uses of `invokedynamic` (lambdas, string
    concatenation, and switch dispatch) into ordinary bytecode

  - Pre-generate lambda-form classes

  - Further incremental CDS improvements

  - Speculative ahead-of-time compilation

- Condensers that require specification changes because they require new
  opt-in constraints:

  - Pre-resolve classes, field accesses, and method invocations (which
    requires the constraint that selected classes cannot be redefined)

  - Non-speculative ahead-of-time compilation (which requires the
    constraint that a pre-compiled class cannot be newly subclassed at
    run time)

  - Eliminate apparently-unused classes and class members (i.e.,
    _stripping_, which requires the constraint that selected classes and
    members cannot be reflected upon)

- New language features that enable further condensation:

  - [Lazy static final fields][lazy]

  - Explicit build-time class initialization

  (Since these are language features, considerable research and
  prototyping will be required.)
