
# Toward Condensers

#### Brian Goetz, Mark Reinhold, & Paul Sandoz {.author}
#### 2023/7/31 {.date}


> We elaborate the concept of _[composable condensers]_ to introduce a
> simple, abstract, immutable, data-driven model of applications so that
> condensers can be expressed as transformers of instances of the model.
> The model is sufficient to express simple condensers; we include two
> examples.

[composable condensers]: 02-shift-and-constrain#condensing-code


The key theme of Leyden is [_selectively shifting and constraining
computation_][02].  _Shifting_ computation is taking work that would
ordinarily be performed at run time and moving it to an earlier phase,
prior to run time, or else to a point later in run time.  We can shift
computation via static transformation (e.g., replacing
`Class.forName("com/foo/Bar")` with `ldc
Constant_Class_info[com/foo/Bar]`) or via dynamic analysis (e.g., running
application code at build time and recording the results for later use).
_Constraining_ computation means narrowing the range of run-time options
that an application would ordinarily have, such as the option to redefine
classes dynamically.  _Selectively_ shifting and constraining means
giving developers the flexibility to choose which sorts of computations
to shift (e.g., compile code ahead-of-time but retain dynamic
deoptimization), and when to shift them (e.g., don't bother compiling
ahead-of-time for ordinary compile-and-test cycles, but do so for
promoted builds).

The primary means for shifting computation is the _condenser_. A
condenser is a component that transforms a program, yielding a program
that is semantically equivalent under a stated set of constraints (e.g.,
"class X will not be redefined"), but may be smaller, faster, or better
suited to a particular execution environment. Examples of possible
transformations include replacing reflective calls with `ldc`
instructions, generating dynamic proxies and lambda proxies
ahead-of-time, propagating and folding constants, stripping dead code,
and compiling to native code ahead-of-time; there are [others][02-eg].
Some of these require no changes to the Java Platform Specification,
whereas others will require additional specification support.

Each condenser will likely perform a narrow, focused task. Developers can
select which condensers to run on their applications, and run different
sets of condensers with different configurations for different build
targets. When running multiple condensers, the output of one condenser
becomes the input to the next. In this way, condensation is similar to a
stream pipeline: It has a source (the initial application configuration),
a number of intermediate stages (condensers that transform the
application), and a terminal stage (which produces artifacts suitable for
use at run time).

The considerations of the intermediate stages are different from that of
the terminal stage, so we define a separate API for condensers than for
packaging an application for final consumption by the JVM.  This document
focuses primarily on the API used by condensers, and the form of data
consumed and produced by condensers.

Whether condensers are ultimately a special kind of `jlink` plugin, or
whether condensation requires tooling separate from `jlink`, is a
question to be resolved later. For now, we will prototype some sort of
"condenser runner" which will run one or more condensers on an
application configuration.

[02]: 02-shift-and-constrain
[02-eg]: 02-shift-and-constrain#research-and-develop-new-condensers-and-new-language-features


## The condenser pipeline

The condensation process starts with a description of the application's
_configuration_, which includes module files, JAR files, class-path
information, configuration metadata (e.g., `--add-module` elections),
application main-class selection, and so forth. The first condenser will
consume the base application configuration and will add, modify, or
remove elements to produce a new configuration. Then the next condenser
will operate on this modified configuration, and so on, until we get to
the terminal stage of the pipeline. We anticipate that the configuration
will eventually include resolved class metadata, heap objects,
ahead-of-time compiled code, and constraints such as "class X cannot be
redefined."

Rather than ask condensers to read class files or other application
configuration from the file system, we constrain condensers to operate on
a _model_ of the application configuration. The initial phase of the
condenser pipeline will _extract_ a model of the application
configuration, and then condensers can take turns operating on the model
as it is passed through the pipeline. The final phase of the pipeline
will _distill_ the model into deployment artifacts, which may be a
modular Java runtime or may be some more-specific deployment format.

![condenser pipeline](images/toward-condensers/fig-1.svg)

The primary action of a condenser will be to augment or modify the
application configuration. Commonly, this will take the form or
generating or modifying application class files, but this may also
include generating or modifying other resources or metadata. Such
transformations can be informed by static analysis of the application as
well as by dynamic analysis, i.e., performing training runs and observing
the application’s behavior.

The API for writing a condenser is deceptively simple:

```
interface Condenser {
    ApplicationModel condense(ApplicationModel model);
}
```

This highlights the fact that a condenser is "just" a functional
transform on the application model -- and therefore condensers can easily
be composed.


## The application model

The application model must be able to represent everything of relevance
about the application being condensed. This includes data and metadata
about class files, modules, symbols, run-time metadata such as class path
and main class or module, semantic-carrying command-line flags such as
`--add-opens`, and so forth.

We adopt the following goals for the `ApplicationModel` API and the
condensation process:

  - _Abstracted away from the representation_ — Condensers should not
    directly read and write files as they do their work; they should
    express their behavior in terms of changes to the model, and let the
    tooling handle the representation.

  - _Explicitly specified_ — The application model and the semantics of
    model elements are strictly defined by the Platform
    Specification. While the model’s schema will evolve over time to
    represent new forms of metadata and constraints, condensers are
    constrained to operate within the model’s schema and respect its
    semantics as defined by the Platform Specification. The application
    model should contain no elements that are not specified by the
    platform, and the APIs should use sealing to enforce this.

  - _Immutable_ — The application model is an immutable representation of
    the application configuration. Condensation is expressed as a
    functional transformation on the application model.

  - _Scrutable_ — It should be possible to answer the question, "what did
    that condenser do?"


### Data model

The data-carrying portions of the application-model API are derived
mechanically from a data model. The data model consists of _entity keys_
and _entity properties_. Keys identify an _entity_ in the model (class,
module, jar, etc.). As a modeling choice, we adopt the discipline that
entity keys contain no actual data other than the identifying
characteristics (i.e., primary key) of the entity; they merely name the
entity in some namespace. All data about an entity (such as the bytes of
a class, or the entries of a JAR) are stored in the entity’s
_properties_, which represent specific facts about that entity.
Properties have a _cardinality_, which constrains how many facts of that
kind there can be for the same entity (a class has one definition; a JAR
can have many entries).  We start with a ridiculously simple model, which
says:

- There is a class path and a list of modules (resolved from the module
  path and the system modules built-in to the environment),
- The class path and modules consist of containers,
- A container has a kind (JAR, modular JAR, or JMOD),
- A container contains classes and resources, and
- Classes and resources have content.

We can visualize this model as a diagram, where ovals indicate entities,
boxes indicate plain data, and arrows indicate properties (decorated by
arity constraints):

![digraph of condenser model](images/toward-condensers/fig-2.svg)

We can translate this model more or less mechanically into an API for
querying the model:

```
sealed interface EntityKey { }

record ModulesKey() implements EntityKey { }
record ClassPathKey() implements EntityKey { }
record ContainerKey(String name) implements EntityKey { }
record ClassKey(ContainerKey container, ClassDesc desc) implements EntityKey { }
record ResourceKey(ContainerKey container, String name) implements EntityKey { }
record ModuleResourceKey(ContainerKey container, String name,
                         ModuleResourceKind kind) implements EntityKey { }

enum ModuleResourceKind {
    CONF, INCLUDE, LEGAL, MAN, LIB, BIN
}

enum ContainerKind {
    JAR, MODULAR_JAR, JMOD
}

interface Model {

  Stream<ContainerKey> modules();
  Stream<ContainerKey> classPath();
  Stream<ClassKey> containerClasses(ContainerKey containerKey);
  Stream<ResourceKey> containerResources(ContainerKey containerKey);
  Stream<ModuleResourceKey> containerModuleResources(ContainerKey containerKey);

  ContainerKind containerKind(ContainerKey containerKey);
  ModuleResourceContents moduleResourceContents(ModuleResourceKey resourceKey);
  ResourceContents resourceContents(ResourceKey resourceKey);
  ClassContents classContents(ClassKey classKey);

}
```

Querying the model can lean heavily on streams; if we wanted to find all
the public interfaces in an application, we can do so as follows:

```
List<ClassDesc> publicIntfs =
    Stream.concat(model.modules(), model.classPath())
    .flatMap(model::containerClasses)
    .map(model::classContents)
    .map(ClassContents::classModel)
    .filter(c -> c.flags().has(ACC_PUBLIC) && c.flags().has(ACC_INTERFACE))
    .map(c -> c.thisClass().asSymbol())
    .toList();
```

What this does is concatenate the module and class-path elements (which
are streams of containers), extract the class entries from these
containers, map each one to a `ClassModel` (defined by the new
[`java.lang.classfile`] API), filter for whether the class represents a
public interface, and finally fetch the symbolic descriptor for each one
of those.

[`java.lang.classfile`]: https://openjdk.org/jeps/8280389


### Transforming the model

The elements of the model, and the model itself, are immutable values.
(This may seem questionable at first, but since we query the model using
streams, which are lazy, we don't want to be mutating the model while
we're in the middle of querying it.) Updating the model is accomplished
by applying an update to it, producing a new model. An update is
effectively a collection of changes (additions, changes, removals) to be
applied. We can ask a model for an updater (which is a builder),
accumulate our changes, and then apply the update to the model, yielding
a new model:

```
interface Model {
    ...
    ModelUpdater updater();
    Model apply(ModelUpdate updater);
}
```

A model updater has a number of mutative methods for adding, replacing,
or removing entity property mappings; it is a builder for a transaction
against the model. Again, the updater API is derived largely mechanically
from the data model:

```
public interface ModelUpdater {

    // modules -> container
    ModelUpdater addToModules(ContainerKey containerKey, ContainerKind kind);
    ModelUpdater removeFromModules(ContainerKey containerKey);

    // class path -> container
    ModelUpdater addToClassPath(ContainerKey containerKey);
    ModelUpdater removeFromClassPath(ContainerKey containerKey);

    // container -> module resource
    ModelUpdater addToContainer(ModuleResourceKey resourceKey,
                                ModuleResourceContents resourceContents);
    ModelUpdater removeFromContainer(ModuleResourceKey resourceKey);

    // container -> resource
    ModelUpdater addToContainer(ResourceKey resourceKey,
                                ResourceContents resourceContents);
    ModelUpdater removeFromContainer(ResourceKey resourceKey);

    // container -> class
    ModelUpdater addToContainer(ClassKey classKey,
                                ClassContents classContents);
    ModelUpdater removeFromContainer(ClassKey classKey);

}
```

Additionally, the model may have integrity constraints, such as "the
container in the class key must match the container it is added to" or "a
container cannot be present as an element of both the modules and class
path". These can be enforced by the model updater. As in the relational
calculus, duplicate identical entity property mappings are considered the
same fact; facts have no independent identity. For multi-value
properties, order of addition is respected, giving them semantics similar
to `LinkedHashSet`.

Obviously this model is limited, but it is already sufficient to
represent many of the condensers that do not require specification
changes. The model and the model API will be extended over time to
support additional entities (e.g., command-line flags), properties (e.g.,
profiling information), and constraints (e.g., "class X cannot be
redefined").


## Modes of operation

Condensers can operate in a variety of modes. The simplest is pure class
file transformation: Enumerate the class files, perform a static analysis
(local or global), and transform some of the class files (or generate new
ones) based on the analysis. Condensers that can operate in this mode
include expanding lambda proxies into classes, or expanding invocations
of dynamic proxies whose interfaces can be identified statically.

Condensation can also involve actually running the code. The `Model` API
provides a means to unpack a model temporarily into a JDK image and start
a JVM using that image with a specified main entry point and command-line
flags. This allows condensers to perform training runs to gather
profiling data or extract program configuration, which can inform the
transformation of classes and the generation of metadata. Examples of
condensers that can operate in this mode are expanding invocations of
dynamic proxies that are instantiated during the training run, and
pre-generating lambda forms.


### Example: Lambda forms

The JDK build process already contains a bespoke condensation process for
pre-generating _lambda forms_, which are internal classes associated with
method handles. This process reduces startup time by selectively shifting
method-handle infrastructure computation from run time to build time.

The JDK build currently builds a preliminary JDK and then runs a minimal
training program that exercises the method-handle infrastructure and
outputs a list of generated lambda-form classes. That list is then fed
into a `jlink` plugin that generates the classes and adds them to the
`java.base` module of the final build of the JDK.

We can implement this process using a condenser and the API described
above; here is the code to run this process as a condenser:

```
// Create the path to the not-yet-condensed JDK build
Path javaHome = Path.of(...);

// Set up the class path for the main class that exercises MH infrastructure
var classPath = List.of(Path.of("make/jdk/src/classes"));

// Set up the module path to the base module of the JDK build
var modulePath = List.of(javaHome.resolve("jmods/java.base.jmod"));

// Initiate the application model from the class and module paths
var model = Condensers.init(classPath, modulePath);

// Create the lambda form condenser with the main class to run
var lfc = new LFCondenser(javaHome,
                          "build.tools.classlist.HelloClasslist");

// Condense to obtain a new application model
model = lfc.condense(model);
```

After condensing we can update the `java.base` module on the file system
from the updated application model. The `LFCondenser` implementation is
also quite simple:

```
@Override
class LFCondenser {

    private final Path javaHome;
    private final String mainClass;

    public LFCondenser(Path javaHome, String mainClass) {
        this.javaHome = javaHome;
        this.mainClass = mainClass;
    }

    public ApplicationModel condense(ApplicationModel m) {

        // Distill the current model to an executable application
        var app = m.distill(location, javaHome);

        // Run with a system property enabling profile output on standard out
        Process p = app.run(List.of("-Djava.lang.invoke.MethodHandle.TRACE_RESOLVE=true",
                                    mainClass));

        // Collect the profile as a string
        String trace;
        try (var in = p.getInputStream()) {
            trace = new String(in.readAllBytes());
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }

        // Get the java.base module
        ContainerKey java_base = m.modules()
            .filter(cK -> m.containerKind(cK) == ContainerKind.JMOD)
            .filter(cK -> cK.name().equals("java.base"))
            .findFirst().orElseThrow();

        // Generate classes from the profile
        Map<String, byte[]> g = SharedSecrets.getJavaLangInvokeAccess()
            .generateHolderClasses(new BufferedReader(new StringReader(trace)).lines());

        // Update the classes in java.base
        ModelUpdater mu = m.updater();
        g.forEach((s, bytes) -> {
            ClassKey classK = new ClassKey(java_base,ClassDesc.ofInternalName(s));
            mu.addToContainer(classK, ClassContents.of(bytes));
        });

        // Apply the updates and return the new model
        return mu.apply();

    }

}
```


### Example: Replacing `Class::forName` with LDC

A simple condenser can scan the contents of all classes present in the
application model, replacing any invocation of `Class::forName`, whose
string argument is a constant value, with an LDC instruction.

```
@Override
public ApplicationModel condense(ApplicationModel model) {
    ModelUpdater updater = model.updater();
    // Stream through all classes
    Stream.concat(model.modules(), model.classPath())
        .flatMap(model::classes)
        .forEach(classKey -> {
            // Obtain the ClassModel for the class and transform it
            // if there are matching Class::forName invocations
            ClassModel cm = matchAndTransform(model.classContents(classKey).classModel());
            if (cm != null) {
                // If transformed update the class contents
                updater.addToContainer(classKey, ClassContents.of(cm));
            }
        });
    return updater.apply();
}

ClassModel matchAndTransform(ClassModel classModel) {
    var modifiedClassModel = new AtomicBoolean();
    // Transform the class model
    byte[] newBytes = cm.transform(ClassTransform.transformingMethods((methodBuilder, me) -> {
        if (me instanceof CodeModel codeModel) {
            boolean m = matchAndTransform(methodBuilder, codeModel);
            modifiedClassModel.compareAndSet(false, m);
        } else {
            methodBuilder.accept(me);
        }
    }));
    return modifiedClassModel.get() ? Classfile.parse(newBytes) : null;
}

boolean matchAndTransform(MethodBuilder methodBuilder, CodeModel codeModel) {
  // Traverse the code model replacing Class::forName invocation instructions
  // with LDC instructions.  Return false if there were no modifications.
  ...
}
```

All of the complexity is encapsulated in the method `matchAndTransform`
that operates on a `CodeModel`, which uses the `transform` method from
`java.lang.classfile`. For each `invokestatic` instruction in the model
the method checks whether the instruction invokes the `Class::forName`
method, and further checks whether the operand on the stack is a constant
`String` value. If so, the method replaces the `invoke` instruction with
an `ldc` instruction referencing a class description whose binary name is
the `String` value, and it removes the instructions associated with
production of the operand if they do not impact other instructions.

In general this kind of condenser -- one that operates on constant values
-- will benefit from consuming a model produced by another condenser that
performs constant folding across the whole application.
