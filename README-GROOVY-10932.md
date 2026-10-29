# GROOVY-10932: pre-compiled trait helpers are loaded instead of read from bytecode

This branch backports [apache/groovy#3076](https://github.com/apache/groovy/pull/3076) to `GROOVY_5_1_X`.
The change is the same as on `master`, written for Java 11, which the 5.1.x line targets: a cast replaces
the pattern-matching `instanceof`. This file only explains the problem and is not part of the fix.

## The problem

When a class implements a trait that is already compiled, `Traits.findHelpers` loads the trait's helper
classes (`Named$Trait$Helper` and the field helpers) through the class loader and builds their `ClassNode`s
by reflection. Every other class on the class path is read from its class file. Loading the helpers has two
effects.

**Incremental compilation fails.** Loading a class makes the JVM resolve the types in the signatures of its
methods. A build tool that recompiles some classes first moves their stale class files aside. A helper whose
methods refer to one of those classes then fails to load:

```
General error during semantic analysis: java.lang.NoClassDefFoundError:
  Unable to configure com.example.Named$Trait$Helper due to missing dependency com/example/Util
```

Gradle tracks this as [gradle/gradle#22814](https://github.com/gradle/gradle/issues/22814).

**The implementer compiles differently.** `Java8.configureClassNode` builds the annotations of the helper's
methods from reflected `java.lang.annotation.Annotation` instances, and reflection reports every member,
defaults included. When the trait is compiled in the same compilation unit, the methods copied into the
implementer carry the annotations as written in the source. When it is pre-compiled, they carry every default
member. `javap -v` on the implementer:

```
-        example.Flag                                  trait in the same compilation unit
+        example.Flag(note="", value=true)              trait pre-compiled
```

The same source compiles to a class with a different binary API depending on what else is compiled. An
incremental build recompiles an implementer against the pre-compiled trait, so its API changes although its
source did not, and every class depending on it is recompiled too.

## Reproducing

- [jdaugherty/groovy-trait-incremental-compile](https://github.com/jdaugherty/groovy-trait-incremental-compile):
  `groovyc-repro.sh` shows the bytecode difference with `groovyc` alone, `demo.sh` in a Gradle build.
- `src/test/groovy/org/codehaus/groovy/transform/traitx/Groovy10932.groovy` covers both effects.

## The fix

A trait read from bytecode is a `DecompiledClassNode`. Its helpers are now resolved through the trait's own
reference resolver (`DecompiledClassNode#resolveClassNullable`), so they are read from their class files like
any other class: classes of the compilation unit first, then the class path. Traits that were loaded as
classes keep the old lookup.

## Verified

- `Groovy10932` passes on this branch.
- In a large Grails 8 application compiled with Gradle's incremental Groovy compilation, the trait helper
  `NoClassDefFoundError` no longer occurs, and every class recompiled against pre-compiled project traits has
  the same binary API as in a full build (12 classes differed before).
