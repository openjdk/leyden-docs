
Project Leyden: Beginnings
==========================

#### Mark Reinhold {.author}
#### 2022/5/20 {.date}


The ultimate goal of this Project, as stated in the [Call for
Discussion], is to address the long-term pain points of Java’s slow
startup time, slow time to peak performance, and large footprint.

In the Call for Discussion I proposed that we address these pain points
by introducing a concept of _static run-time images_ to the Java Platform,
and to the JDK.

  - A static image is a standalone program, derived from an application
    and a JDK, which runs that application -- and no other.

  - A static image is a _closed world_ with respect to the classes that
    it can load: At run time it cannot load classes from outside the
    image, nor can it create classes dynamically.

The closed-world constraint imposes strict limits on Java’s natural
dynamism, particularly on the run-time reflection and class-loading
features upon which so many existing Java libraries and frameworks
depend.  Not all applications are well suited to this constraint, and
not all developers are willing to live with it.

So rather than adopt the closed-world constraint at the start, I propose
that we instead pursue a gradual, incremental approach.

We will explore a spectrum of constraints, weaker than the closed-world
constraint, and discover what optimizations they enable.  The resulting
optimizations will almost certainly be weaker than those enabled by the
closed-world constraint.  Because the constraints are weaker, however,
the optimizations will likely be applicable to a broader range of
existing code -- thus they will be more useful to more developers.

We will work incrementally along this spectrum of constraints, starting
small and simple so that we can develop a firm understanding of the
changes required to the Java Platform Specification.  Along the way we
will strive, of course, to preserve Java’s core values of readability,
compatibility, and generality.

We will lean heavily on existing components of the JDK including the
HotSpot JVM, the C2 compiler, application class-data sharing (CDS), and
the `jlink` linking tool.

In the long run we will likely embrace the full closed-world constraint
in order to produce fully-static images.  Between now and then, however,
we will develop and deliver incremental improvements which developers can
use sooner rather than later.

<p class="br">Let us begin!</p>


[Call for Discussion]: https://mail.openjdk.java.net/pipermail/discuss/2020-April/005429.html
