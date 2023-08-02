
# Condensing Indy Bootstraps

#### Brian Goetz {.author}
#### 2023/8/2 {.date}


One of the targets of condensation is bootstraps for `indy` (and `condy`);
shifting the work of these bootstraps from first use to build time.  (Note that
a bootstrap _already_ represents computation that has been shifted from build
time to run time, so condensation is effectively "unshifting" this work.)  Not
all bootstraps will be amenable to shifting back to build time; some may depend
on information or operations only available at runtime. However, the bootstraps
used by `javac` generally avoid runtime dependencies the most dynamic features
of method handles, and therefore are likely candidates for such re-shifting.

We have thus far explored two directions for how a condenser would shift
bootstraps to build time: having a hand-built condenser for each well-known
bootstrap, and build-time evaluation of bootstraps.  In this document we present
a third alternative: building bootstrap shifting more directly into the
programming model.

## Approach #1: Bespoke condensers

The most obvious approach is to write a condenser for each bootstrap that we
want to shift.  This has the advantage that bootstraps have a specification, so
the condenser can lean on that specification to ensure a semantic-preserving
conversion.  Writing such condensers is usually straightforward, and there are
not that many bootstraps to do this for.

While this seems like a reasonably bounded amount of work to "get the job done",
there is a price.  Aside from the work of writing a separate condenser for each
bootstrap, it ends up creating technical debt for future maintenance. Having a
separate bootstrap and condenser means we have _two separate implementations
using two different programming models_ of each bootstrap, one for online use
and one for offline -- and they must be kept in sync going forward.
Unfortunately this is going to be difficult; not only is it easy to update one
and forget to update the other, but even when updating them in tandem, the
conceptual gap between the programming models used by each virtually guarantees
that subtle bugs will slip in.  As the bootstraps are maintained, costs are
increased along with the risk that the two implementations accidentally diverge.

## Approach #2: Build-time evaluation of bootstraps

At the other end of the spectrum, it seems tempting to actually try to run the
bootstrap at build time, and then reverse engineer the linkage back to bytecode.
(This is a constrained form of build-time evaluation.)  This runs into two
issues, one semantic and one practical.

The semantic issue is that not all bootstraps are amenable to this sort of
transformation; linkage may depend on the runtime environment, sometimes in
subtle ways.  At the very least, we would want bootstraps to "opt in" to such
treatment, but developers still have to understand they're opting into, and this
is where it gets subtle: there's a long list of things not to do here.  Worse,
as you dive into the details of this list, things get "squishy" fast. Specifying
what operations a shiftable bootstrap could perform is a significant and
imprecise exercise, and it will likely be easy for bootstrap developers to
accidentally stray outside of the lines.  For example, while it is tempting to
say a bootstrap should be "side-effect free", most bootstraps do have some
benign side-effects, such as logging or dumping generated code to disk for
debugging.  The same is true for nearly every constraint one could imagine;
there are exceptions and exceptions-to-exceptions for each.  This puts bootstrap
developers in the position of judging whether a given side-effect (or impurity
or mutability or identity dependence or ...) is "benign" or not.

Assuming the semantics are amenable, there are still technical challenges.
Tooling would have to inspect a `CallSite` and its `MethodHandle` target and
reverse engineer them back to bytecode.  This has additional challenges, some of
which are just a matter of "work", and others that are more subtle.  We would
need a mechanism for reflecting over method handles, since the linkage target
may not always be a `DirectMethodHandle`.  Bootstraps often load hidden classes;
we would need to be able to intercept loading of such classes by bootstraps, or
reflect over those hidden classes, so that they can be captured and stored in
the archive.  But even if we had these mechanisms, there are still challenges.
For example, the `insertArguments` combinator can curry live objects onto method
handles.  Sometimes this is harmless; the object might be a constant like a
string, which is easy to reduce back to bytecode.  Even if it is an object with
significant identity and even mutability (such as dispatch caches for pattern
switches), this might still be harmless if it is the only outstanding reference
and can't escape.  But the devil is deep in the details.

The basic problem is that indy linkage is so flexible and expressive that it can
be very difficult to look at a resulting method handle chain and be confident we
understand the environmental dependencies.  While this path is not impossible,
much complexity and brittleness lies in this direction; it amounts to
"unscrambling the egg", which at best is going to be messy and imprecise.

## A third option

If offline evaluation requires us to unscramble an egg, the alternative is to
not scramble the egg in the first place.  Some bootstraps have semantics that
are inextricably tied to runtime state; other bootstraps have semantics that are
amenable to shifting, and can work either "online" or "offline".  We can avoid
the brittleness of both the bespoke condenser approach and the build-time
evaluation approach by bringing this notion of "shiftable bootstrap" _into the
programming model_.

While `indy` bootstraps are ordinary static methods which can be theoretically
invoked directly with arbitrary arguments, when they are invoked by the JVM to
link an `indy` site, they are always invoked with objects that are the result of
loading constants from the constant pool, such as `String`, `Class`,
`MethodHandle`, etc.  (We call these _live objects_ to differentiate them from
their unresolved symbolic form in the constant pool (JVMS 5.4.3.6).)  Similarly,
a bootstrap produces a "live" result in the form of a `MethodHandle` (wrapped
with a `CallSite`), and access control is mediated through a `Lookup` (a live
object representing an access control context).

If we want to represent offline evaluation of bootstraps, the live objects used
by the online evaluation are not necessarily available; what is available is the
symbolic information present in the constant pool.  In JDK 12 we introduced the
`java.lang.constant` API, which can symbolically describe all the kinds of
loadable constants using the `ConstantDesc` hierarchy (`String` and friends were
retrofitted to implement `ConstantDesc` directly; new implementations were
created for `ClassDesc` and friends).  An "offline bootstrap" would deal not in
live objects like `Class`, but symbolic descriptors like `ClassDesc`.

Each kind of loadable constant has both a live and symbolic form (e.g., `Class`
and `ClassDesc`), and there are means for converting between them, but there are
limitations in both directions.  For example, hidden classes have a `Class` but
cannot be described with a `ClassDesc`, so not all `Class` objects have a
symbolic description.  In the other direction, converting from a `ClassDesc` to
a `Class` requires the aid of a `Lookup`, and the corresponding `Class` must be
accessible to that lookup and loadable by the appropriate class loader.
Shiftable bootstraps would have to be able to work within the limitations of
both domains.

We can represent a shiftable bootstrap with two entry points, one accepting and
producing the traditional live objects, and the other accepting and producing
symbolic descriptors.  In reality, what this means is that such bootstraps will
usually operate by classfile generation, and the two entry points are mostly
concerned with unpacking the arguments into a common form so the two entry
points can share the main code generation logic.  On the other side of the code
generation, the two entry points may have different paths for "unpacking" the
result of class generation (loading it vs storing it in the archive).  Access
control decisions would be made through the ordinary mechanism of executing the
generated bytecodes.

### Paired bootstraps

An `indy` bootstrap is a static method with three standard metadata arguments
(lookup, name, and type) as well as additional bootstrap-specific arguments.
Bootstraps can be "varargs" methods and the JDK will adapt the shape of the
static argument list with the shape of the bootstrap appropriately.  Bootstraps
generally expect their arguments will be loadable constants, since the static
argument list for the bootstrap generally originates in the `BootstrapMethod`
attribute of the classfile (though bootstraps are ordinary methods and can be
invoked directly as well).

Since bootstraps form an ABI, we can't change the signatures of existing
bootstraps, but we can pair existing bootstraps (which are designed for "online"
use) with their offline siblings.  We can mechanically translate the argument
list of an "online" bootstrap into the corresponding "offline" one by
translating `Class` to `ClassDesc`, `MethodType` to `MethodTypeDesc`,
`MethodHandle` to `MethodHandleDesc`, leaving `String`, `Integer`, `Long`,
`Float`, and `Double` alone, and translating everything else to `ConstantDesc`.
We can then annotate the online bootstrap to indicate that it has an offline
sibling that can be used to condense away an `indy`.  For example, the
bootstraps from `LambdaMetafactory` are:

```
@OnlineBootstrap
public static CallSite metafactory(MethodHandles.Lookup caller,
                                   String interfaceMethodName,
                                   MethodType factoryType,
                                   MethodType interfaceMethodType,
                                   MethodHandle implementation,
                                   MethodType dynamicMethodType)
        throws LambdaConversionException { ... }

@OnlineBootstrap
public static CallSite altMetafactory(MethodHandles.Lookup caller,
                                      String interfaceMethodName,
                                      MethodType factoryType,
                                      Object... args)
        throws LambdaConversionException { ... }
```

We tag these with `@OnlineBootstrap` to identify that these are methods intended
to be used as bootstraps for which an offline sibling exists; the offline
sibling has a signature mechanically derived from the online bootstrap
signature.  We replace the `Lookup` argument with a `ClassDesc` describing the
host class.  The bootstrap returns not a `CallSite`, but a symbolic descriptor
for the desired (constant) linkage:

```
@OfflineBootstrap
public static OfflineIndyLinkage   metafactory(ClassDesc lookupClass,
                                               String interfaceMethodName,
                                               MethodTypeDesc factoryType,
                                               MethodTypeDesc interfaceMethodType,
                                               DirectMethodHandleDesc implementation,
                                               MethodTypeDesc dynamicMethodType)
        throws LambdaConversionException { ... }

@OfflineBootstrap
public static OfflineIndyLinkage   altMetafactory(ClassDesc lookupClass,
                                                  String interfaceMethodName,
                                                  MethodTypeDesc factoryType,
                                                  ConstantDesc... args)
        throws LambdaConversionException { ... }
```

### Condenser operation

There would be a single condenser for condensable `indy` bootstraps.  It
searches for `indy` sites, and inspects the bootstrap for the `@OnlineBootstrap`
annotation. If it finds one, it looks for the offline sibling bootstrap (by
mechanical signature mapping), loads the bootstrap class, and invokes the
offline bootstrap.  The result symbolically describes a `OfflineIndyLinkage`,
which would encode the result of the linkage, including which hidden classes are
generated as a result of linkage, and the owner, name, and type of the target
method. The condenser adds the new classes to the application configuration as
nestmates of the class holding the `indy`, and replaces the indy invocation with
a bytecoded invocation of the target method.

### What about condy?

Condensing `Constant_Dynamic_info` ("condy") has additional complexities over
condensing `indy` because `condy` may appear in more places, and may take on
arbitrary types.  A condy can appear as the operand of an `LDC`, in the static
argument list of an `indy` or `condy` bootstrap, or in any attribute that uses
constant pool indexes.

The approach outlined here for `indy` can likely be extended to condy, with
limitations.  These will be explored in a later iteration.

### Was using `indy` in translation a mistake?

It may be tempting to conclude from all this that it was a mistake to shift
linkage from build time to runtime using `indy` in the first place.  But there
were good reasons we chose to translate lambdas with `indy`, and those reasons
still hold today.

An `indy` site that captures a lambda or method reference captures the desired
semantics explicitly and minimally; a statically generated lambda proxy class is
full of accidental detail that obfuscates the programmer intent, and is
therefore harder to optimize.  Deferring linkage to runtime allows code
generation to be improved by simply updating the JDK rather than requiring
recompilation, and allows a multiplicity of code generation strategies that can
be selected at runtime (as is done by the string concatenation bootstrap).  Not
all programs want to optimize solely for startup time.

### Writing shiftable bootstraps

The upshot is that as we adjust the programming model for bootstraps, the
equilibrium point shifts to reward bootstraps that can run either online or
offline.  This almost certainly means such bootstraps will run by bytecode
generation, with two shallow prefixes that map the live or symbolic inputs to a
common format for code generation, and two shallow exit points that take the
generated code and either load it or package it for the condenser's consumption.
The existing bootstraps in the JDK, such as `LambdaMetafactory`, can be
relatively easily refactored into this "two-headed" form.

Some bootstraps may have constant linkage but that linkage includes mutable
state; the bootstraps for pattern switch classification may want to use this
technique (so that we can cache dispatch decisions for later reuse).  With
traditional online bootstraps, this is usually done today by currying a mutable
object onto the method handle with `insertArguments`, but this is just as easily
done with code generation -- the static method that replaces the `indy` can
instantiate a new instance of the required state carrier and pass that
internally to a factory or constructor.
