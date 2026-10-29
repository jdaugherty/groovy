# A closure's return type signature depends on what else is compiled

This branch fixes a difference in the bytecode of statically compiled closures. It is reported as
[GROOVY-12550](https://issues.apache.org/jira/browse/GROOVY-12550). This file only explains the problem and is not part of the fix.

## The problem

With static compilation, the `doCall` method of a closure is declared with the closure's inferred return type.
When that type is a parameterized type whose class is in the same compilation unit, its type arguments are
dropped. When the class is read from the class path, they are kept:

```groovy
interface Up {}
class Res<T> { List<T> items = [] }

@groovy.transform.CompileStatic
class Svc {
    static <V> V timeIt(Closure<V> body) { body.call() }
    def <U extends Up> Res<U> read(Class<U> type) { new Res<U>() }
    Res<Up> concrete(Class<Up> type) { timeIt { -> read(type) } }
}
```

```
Svc compiled together with Res:     public Res doCall()
Svc compiled against a compiled Res: public Res<Up> doCall()
```

Reproduced with Groovy 4.0.33, 5.1.3 and `master`. A standalone reproducer is at
[jdaugherty/groovy-closure-return-type-signature](https://github.com/jdaugherty/groovy-closure-return-type-signature).

The same source compiles to a closure class with a different signature depending on what else is compiled.
An incremental build that recompiles `Svc` without `Res` produces a closure class whose binary API differs
from the full build, so compile avoidance treats it as changed although its source did not change.

The pattern occurs in Grails applications: `@Transactional` and similar transformations move a method body
into a new method without the original method's type parameters, so a closure there that used to return `Res<U>`
now returns a concrete `Res<Up>`.

## The cause

`ClosureWriter.createClosureClass` normalises the inferred return type with:

```java
else if (returnType.isPrimaryClassNode()) returnType = returnType.getPlainNodeReference();
```

`isPrimaryClassNode()` is true for the class node of a class in the compilation unit and for every reference to
it, including a parameterized one such as `Res<Up>`, and `getPlainNodeReference()` drops the type arguments. A
reference to a class read from bytecode is not primary, so it keeps its type arguments unless they are
unresolved. The branch came with GROOVY-9971.

## The fix

Only the class node itself, which carries the declared type parameters rather than type arguments, is replaced
by its plain reference. A parameterized reference to a class of the compilation unit goes through the same check
for unresolved generics as any other type, so both builds produce `Res<Up> doCall()`. That is the signature a
closure already gets for a parameterized library type such as `List<String>`, so only closures returning a
parameterized class of their own compilation unit change.

On `master`, `ClosureWriter.inferredClosureReturnType`, which normalises the return type the same way for
packed closures, gets the same change.

## Test

`src/test/groovy/org/codehaus/groovy/classgen/asm/ClosureReturnTypeSignatureTest.groovy` compiles the example
once in a single compilation unit and once against a compiled `Up` and `Res`, and compares the descriptors and
signatures of each closure's `doCall` method. It fails without the fix.

## Verified

- `ClosureReturnTypeSignatureTest` fails without the fix and passes with it, on `master` and on `GROOVY_5_1_X`.
- The full core test suite passes on `master` (18796 tests) and on `GROOVY_5_1_X` (13482 tests).
