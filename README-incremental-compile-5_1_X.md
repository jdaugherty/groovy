# Groovy 5.1.x with the fixes for incremental compilation

This branch combines three fixes on top of `GROOVY_5_1_X`. Each makes Groovy compile the same source to the same
class file whatever else is in the compilation unit, which incremental builds depend on. Each fix has its own
branch, whose README explains the problem:

| Fix | Branch | Upstream |
|---|---|---|
| GROOVY-10932: read pre-compiled trait helpers from bytecode | [`GROOVY-10932-5_1_X`](https://github.com/jdaugherty/groovy/blob/GROOVY-10932-5_1_X/README-GROOVY-10932.md) | [apache/groovy#3076](https://github.com/apache/groovy/pull/3076) |
| GROOVY-12544: `InnerClasses` entries for nested classes outside the compilation unit | [`GROOVY-12544-5_1_X`](https://github.com/jdaugherty/groovy/blob/GROOVY-12544-5_1_X/README-GROOVY-12544.md) | [apache/groovy#3077](https://github.com/apache/groovy/pull/3077) |
| Keep the type arguments of a closure's inferred return type | [`closure-return-type-signature-5_1_X`](https://github.com/jdaugherty/groovy/blob/closure-return-type-signature-5_1_X/README-closure-return-type-signature.md) | [GROOVY-12550](https://issues.apache.org/jira/browse/GROOVY-12550), `master` fix on [`closure-return-type-signature`](https://github.com/jdaugherty/groovy/blob/closure-return-type-signature/README-closure-return-type-signature.md) |

This file is not part of any fix.

## Building it

With JDK 17 or later:

```
./gradlew publishToMavenLocal -x :groovy-binary:publishToMavenLocal -x javadoc -x groovydoc
```

This publishes every module to `~/.m2` as `5.1.4-SNAPSHOT`.

## Using it in a Gradle build

An init script switches a build to the local snapshot without changing it. It applies to the root build only,
so an included build that compiles against Gradle's own Groovy keeps its version:

```groovy
// custom-groovy.init.gradle, used with: ./gradlew -I custom-groovy.init.gradle ...
if (gradle.parent == null) {
    settingsEvaluated { settings ->
        settings.dependencyResolutionManagement.repositories {
            mavenLocal { content { includeGroup 'org.apache.groovy' } }
        }
    }
    allprojects {
        configurations.configureEach {
            resolutionStrategy.eachDependency { details ->
                if (details.requested.group == 'org.apache.groovy') details.useVersion('5.1.4-SNAPSHOT')
            }
        }
    }
}
```

## What it changes in a real build

In a large Grails 8 application, a change to a method body of a utility class makes Gradle's incremental Groovy
compilation recompile 541 classes of the module that contains it. With Groovy 5.1.3, 12 of them came out with a
different binary API than in a full build, because they implement pre-compiled project traits, so the downstream
module recompiled about 12,000 classes. With this branch and the Gradle change in
[gradle/gradle#39622](https://github.com/gradle/gradle/pull/39622), none of the 541 differs and the downstream
module is up to date.

The remaining differences in that application come from Grails' own compile-time transformations, which need a
fix in Grails.
