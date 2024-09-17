# Thoughts on Training Runs

## Dan Heidinga {.author}
## 2024/9/3 {.date}

## Training Runs
The JVM uses observation to drive optimization.  It records the actions the current program executing is actually doing - which paths are executed, which methods are called, which types are observed - and uses that to optimize the "hot spots" in the program. Many of these optimizations are speculative in that they assume some condition that has held so far in the program will continue to hold true, and generate code for that condition while also providing compensation code to fall back if the condition becomes false.

Training runs are a way of observing what an application is doing across *different runs*.  The same kind of profiling information is recorded in one instance of the application and is preserved for use to optimize a subsequent execution of the program.  This allows the same kind of speculative optimizations the JVM does at runtime to be *shifted* earlier to later executions can be optimized further, providing better startup and warmup of the application.

Static analysis is an alternative approach which attempts to determine which methods to optimize without running the program. It is less effective than observing what the program is actually doing, especially for a highly dynamic language like Java.  Observing the program provides greater knowledge of the actual behaviour.

The Leyden project has found training runs to be unreasonably effective at driving startup and warmup optimizations.

## Types of training runs
There are primarily two types of training runs:

* integration tests - which run at build time
* production workloads - which require training in production

Note, training runs are currently focused on the 3 built-in classloaders and the classes loaded by them.  Changing the classes will require retraining.


## The three phases
The current Leyden workflow involves three stages:

* *training run* where the application is run while being observed
* *assembly phase* where the AOTCache is assembled from the training run data
* *production run* where the AOTCache is deployed to speed up startup and warmup.

There are three phases to avoid accidentally capturing unintended state from the training run into the AOTCache.  The AOTCache stores loaded and linked classes, profiling data, and generated code.

By separating the training and assembly phases, we ensure that the assembly phase is a "clean slate" by definition - there is no accidental state that can be entangled with the AOTCache.

This requires that the classes that are loaded and linked in the training run are the *same* classes that are loaded and linked in the assembly phase.  Including any hidden classes generated in response to i.e. Lambda generation by the LambdaMetaFactory.  To ensure that the classes are the same, the VM needs to not only know the class name:class loader relationship, it also needs to know that the class loader will reliably produce the same class for a given name.  Currently, this can only be asserted for the three built-in classloaders: System, Extension, & Boot loaders.  Future work may extend this to other loaders as well.

## Leyden's current approach
Leyden's current approach ensures that the classes used and the training data - such as method profiles - match exactly.  This sidesteps needing heuristics for how applicable the training data is as the class base and observations are from the same run.  Hidden classes, which are generated at runtime, are the most difficult to map in this model as they don't have stable names.  Instead, they can be tracked by knowning the `inputs : generated class` mapping for a given class generator.  This way the training data for a hidden class named i.e. `Foo/0xACD00` from the training run can be applied to the recreated hidden class named `Foo/0xCF00` as we know they were created from the same inputs.

Profiles are from a single training run in the current approach and are not merged.  Future work may investigate alternative ways to capture training data and even alternative patterns.

## Training runs and deployments
There are many ways training runs could be operationalized.  Let's explore a few different options that can be done today or may be supported in the future.  It's important to keep in mind the distinction between training runs and deployments when reading this section.  Many problems that seem like they could, or even should, be solved by training runs are actually better viewed as deployment concerns.

Training runs as stated above should be viewed as fairly simple observations of a workload converted into an artifact (the AOTCache) - how those observations are made at scale and how the artefacts are put in production are more deployment concerns best solved outside the JVM.


### Single shot training
In single shot training, the application is run once to generate the training data, which is used to assemble the AOTCache.  The production runs than experience faster startup and warmup by using the AOTCache.  The same AOTCache can be used on the same machine or deployed to many machines.  The key characteristic of this training style is the single training run used for potentially many deployments.

Training can be done during a the build process:

* by using integration tests as a proxy for the application
* by starting the application and having it run to a certain point or serve dummy requests

Training can alternately be done by in production:

* by using "canary deployments" where the application is deployed in a training mode, generates the AOTCache, and subsequent deployments use the canary's AOTCache to optimize their deployments.

### Linear training (cascade)
Linear training is useful if you want to be able to train the application more than once.  The key constraint in the current Leyden approach is that the class base must be kept constant - JEP XXX requires that preloaded and linked classes must be loaded together to provide stable identities - and current training data is tied to those stable identities.

In linear training, the application is trained, an AOTCache is assembled, and then the application uses the AOTCache and generates new training data to assemble a second cache.

```
Train{no cache} -- AOTCache1 --> Train{AOTCache1} -- AOTCache2 --> Deploy{AOTCache2}
```

Possible benefits of linear training include:

* more classes included in the AOTCache as more may be loaded in subsequent runs
* more profile data and AOT code may be available as more methods execution will have run

Linear training may allow more classes or we may decide to keep the class base constant and only allow additional profiling / aot code to be generated. More investigation required.

### Merging training runs
One of the constraints of the current Leyden approach is that the class base is kept constant to allow for classes to have stable identities.  These stable identities unlock the pre-loading and linking and make it easier to associate classes to their profile and aot code.

We have so far avoided working on merging training runs due to the complexity it adds - once the class base may differ between two runs it is hard, especially for hidden classes, to assert the profiles observed in two different runs actually apply to the same classes.  Never mind trying to make that assertion across class bases that have upgraded libraries and are actually different.

#### Canary deployments
A common devops deployment strategy is to use canary deployments where an updated version ("canary instance") is rolled out to a small number of servers and used to serve production requests.  This allows the update to be observed and tested in production before rolling it out to all servers.  If there is a problem, the canary instances are shutdown and the rollout doesn't occur.

Canary instances are an excellent source of training data.  They run production workloads in production environments and can be observed.  This is an ideal place to do the training.

Once trained in a canary deployment, the trained instances can be redeployed with improved startup time to a small number of instances as a validation cycle before rolling out to the whole production fleet.

Canary deployments could use single shot training with training only occurring on the canary instance, or they could use linear training and have canary instances start with training data from the integration tests / build system.

### Max operator
Even though we do not currently aim to merge training, we can still benefit from multiple distinct training runs.  By doing training on *N* instances and producing *N* AOTCaches, we can observe which trained instance produces the best - or most stable - performance and deploy that instance across the full set of machines.  The "winning" version acting as though a `max(a,b,c,...,n)` operator was used to pick the best deployed version.  The limits here are not in the JVM but rather in the deployment infrastructure.

### Layered training runs
Docker images use a layered filesystem where each layer can add or overwrite an existing file.  This allows deployment containers to be built up based on how frequently they change - with separate layers for the JDK runtime, the application framework, and finally the user's application code.

It may also be beneficial to allow AOTCache's to be layered with each layer adding new content to the cache.  There is a lot of difficultly in doing this as it would add new classes, new class resolution statuses across layers, as well as potentially new heap objects (at minimum, the class mirrors).  Never mind the complexity of multiple sources of profiling and AOT code.

It's something to keep in mind but is not a current design priority.

### Cost of training data collection
Ideally, the cost of running training should be so minimal that users are willing to always run with training enabled.  This way they could treat any instance as a training instance and use some mechanism to request the current VM trigger an assembly phase producing a new AOTCache.

There are various options for how the mechanism to trigger the assembly phase should work, but to get the widest possible use training must be cheap enough that users don't see a cost to always enabling it.

Training includes the recording the classes, the resolved cp entries, the Method Data Objects (MDO), and additional data used by the compiler - such as the initialized classes - when compiling.  Recording this must take minimal time and minimal memory.

### Autotraining
If as proposed above, the cost of training is minimal, then we could enable autotraining where any instance which doesn't have an AOTCache would automatically create one on first use.  These automatic AOTCaches could be stored in either user-specific directories or in `/tmp` as they need not be entirely persistent.  Training again and recreating them should have minimal cost.  This is likely of most use for local deployments.

### Profiles & AOT vs preloaded classes
While layered training and linear training touched on this, it's good to be explicit.  The preloaded and prelinked classes are the common base of all other Leyden premain optimizations.  It may make sense, in some deployment scenarios, to separate the generation of the class part from the profile collection and AOT generation.

Determining the set of classes to load - and doing so at build time - is a simpler problem than collecting production profiles.  This is why it may make sense to separate the steps.

Note that this could also be addressed by Linear training and may not require separation.  Often many ways to solve the same problem.

### Externalize vs internalize training data
Leyden training data is currently only stored in the AOTCache.  There is no external form that tools and other processors can read, manipulate, or generate.  While this feels like a potential limitation, it may actually be another deployment concern rather than a JVM limitation.

Given the class base must be kept constant, the training data (profiles and compiler info) is only relevant in conjunction with those exact classes.  This means it may be reasonable to create tools that can read the AOTCache, but is unlikely to be worth converting the hard-won stable class identities back into symbolic information.

Time will tell on what is needed here as we gather more use cases for how externalized data could be used.

### Assembly phase: the hidden cost
There is another cost we haven't talked about yet that affects deployment more than training but should be considered when determining whether to externalize the training data or not.  And that cost is running the assembly phase.

Our current workflow is 3 phases: training, assembly, and production.  The assembly phase doesn't run the application but it does essentially create another instance of the JVM in memory.  We've been working on moving from our 3 phase workflow to an integrated workflow where the assembly phase occurs in a forked VM launched when training ends.

In this integrated workflow, any deployment that is used for training may also be used to execute the assembly phase, which would increase the resources needed by deployment (now running two instances of Java rather than one).

If deployments typically have enough "head room" in their resource allocations, this is a non-issue.  If they don't, this may affect where and how training can be done.  For local deployments (desktop applications) the integrated workflow is a win.