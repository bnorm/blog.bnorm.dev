---
layout: post
title: Exploring Kotlin IR
description: Exploring Kotlin IR
tweet: https://twitter.com/bnormcodes/status/1226929535620390912?s=20
---

> At the time of writing this article, Kotlin IR is experimental. As such, information contained in
> this article about IR could be out-of-date or incorrect. If official documentation for Kotlin IR
> exists, please refer to it first.

Ever since learning about them, I’ve been very interested in Kotlin compiler plugins. Even the
[limited list of official supported plugins][official-plugins] hint at the potential available. A
plugin like `kotlin-serialization` shows how it is possible to generate code for [marshalling] a
Kotlin class. A plugin like `allopen` shows it’s possible to transform classes to be non-final at
runtime. It is Java annotation processing; but with more power.

My adventure into this world started a few months ago when I once again looked longingly at a Groovy
language feature called [Power Assertions][power-assert]. If you are unfamiliar with this language
feature, [it was original developed by the Spock testing framework, and introduced in Groovy 1.7]
[power-assert-history]. Its goal is to show the value of everything involved in the assertion which
failed.

```groovy
a = 10
b = 9
 
assert 91 == a * b

// Output:             
//
// Exception thrown
//
// Assertion failed: 
//
// assert 91 == a * b
//           |  | | |
//           |  10| 9
//           |    90
//           false
//
//     at ConsoleScript2.run(ConsoleScript2:4)
```

This context as to why the assertion failed is indeed something to desire. Many libraries in the JVM
ecosystem (AssertJ, Hamcrest, Truth, Kluent, Atrium, Strikt, etc) provide meaningful error messages
but they are usually limited to the known types of the library. Many of these libraries have ways to
provide custom assertion extensions so you can provide something meaningful when testing your own
classes. But I’ve always seen this as boiler plate and found it tedious to write.

# Introducing: kotlin-power-assert

[kotlin-power-assert] is a Kotlin compiler plugin which finds every call to `kotlin.assert(...)` and
transforms it to include all the expressions in the assertion in the error message.

```kotlin
val hello = "Hello"
assert(hello.length == "World".substring(1, 4).length) { "Incorrect length" }

// Output:
//
// java.lang.AssertionError: Incorrect length
// assert(hello.length == "World".substring(1, 4).length)
//        |     |      |          |               |
//        |     |      |          |               3
//        |     |      |          orl
//        |     |      false
//        |     5
//        Hello
//         at <stacktrace>
```

This is done via an `IrGenerationExtension` which is a specific kind of extension to the Kotlin
compiler. From this extension point, you are given access to the IR layout of the file being
compiled and are allowed to transform it before the Kotlin compiler continues to the next phase of
compilation.

If you want to learn more about the architecture of Kotlin compiler plugins I highly recommend
watching [Kevin Most’s presentation from KotlinConf 2018][kevin-most-kotlinconf-2018] which goes
into details not covered here.

# Kotlin IR

Kotlin IR, or Kotlin Intermediate Representation, is the new internal representation used by the
Kotlin compiler when it parses Kotlin source files. It uses this representation to than perform a
series of ["lowerings"][ir-lowerings] which transform the code.

These lowering operations include [tailrec transformation][ir-lower-tailrec], [string concatenation
to StringBuilder][ir-lower-concat], [suspend function transformation][ir-lower-suspend], and
[for-loop optimizations][ir-lower-loop]. These lowerings are then performed in phases, with more
general transformations performed first. Once all lowerings are complete, each compiler backend, one
for each supported platform (JVM, JS, or Native), takes the IR and translates it into the platform
specific representation.

By using an intermediate representation, the Kotlin compiler is able to share lowerings across all
the compiler backends. It also allows developers to write a single compiler plugin which will work
on all platforms. Otherwise a plugin which wants to support all Kotlin platforms would need to write
a transformation for each platform specific representation.

# Transfroming Kotlin IR

Kotlin IR is represented as a tree of elements. To navigate, you can use an `IrElementVisitor` which
will visit each `IrElement`. If you want to transform the code, you can use an
`IrElementTransformer` which will allow you to manipulate the elements visited and even return a
completely different element in place of the one being visited. To create an IR tree of your own,
you can use the [many builder functions][ir-builders] available.

As Kotlin IR is still experimental, only a few of the official compiler plugins have been converted
to also support IR. However, many lowerings are currently available which provide good examples to
explore.

# Transforming assert() Calls

So with a little background on Kotlin IR, how is the assertion function transformed to include
additional information?

It all starts with an `IrElementTransformer` which navigates the IR tree and visits each instance of
an `IrCall`. This represents every call to any function. If the function being called is `assert()`,
then the call needs to be transformed.

Next, the IR tree is navigated from that point, finding every instance of an `IrExpression`. An
expression is anything that can return a value: function calls, variable/field access, constants,
if/when expressions, etc.

Then every `IrExpression` found is moved to a temporary variable (via `irTemporary` builder) and
replaced with an access expression to the temporary variable (via `irGet` builder). This allows the
error message to access the same temporary variables used by the assertion condition.

Once the assertion expressions are transformed, the assertion call is replaced with a series of `if`
statements (using `irIfThen` builder) and throw statements (using `irThrow` builder). The
constructor call to `AssertionError` can then include all the temporary variables, formatted
appropriately and include the source code of the original `assert()` call.

Finally, to get the source code for the original `assert()` call, start with getting the source code
for the entire file. This can be accessed through visiting the `IrFile` and using the path extension
function to get the `path` to the source file being compiled. From there, source code range
information can be retrieved from each `IrElement` using the `fileEntry` from the parent `IrFile`.

```kotlin
override fun visitFile(declaration: IrFile): IrFile {
  file = declaration
  fileSource = File(declaration.path).readText()
  ...
}

override fun visitCall(expression: IrCall): IrExpression {
  val callSource = fileSource.substring(expression.startOffset, expression.endOffset)
  val callIndent = file.fileEntry.getSourceRangeInfo(expression.startOffset, expression.endOffset).startColumnNumber
  ...
}
```

Putting all this information together, the expression temporary variables are sorted based on start
column and included in the assertion message string with some "fancy" formatting (using `irConcat`
and `irString` builders).

Unfortunately, in reality it is more complicated that what was just described. Boolean expression
which include `&&` or `||` tend to make things more complicated due to [short-circuiting]. But
hopefully this simplification helps you understand what is going on behind the scenes.

# Writing Your Own Plugin

If you have an idea for a Kotlin compiler plugin, the first thing I recommend is to use the
fantastic library [kotlin-compile-testing]. This library will allow you to compile code in unit
tests and to even load the compiled code so you can test it’s behavior.

The second thing I recommend is to set your expectations correctly. As Kotlin IR is still
experimental, there are still a number of things which do not work well. It could be a while before
Kotlin IR is the default mode for the Kotlin compiler. Make sure you are comfortable with these
risks before using or writing a compiler plugin based on IR.

And finally, clone and load up the Kotlin git repository. Being able to navigate through all the IR
code and lowerings was incredibly useful to find examples on how to perform different
transformations.

[official-plugins]: https://github.com/JetBrains/kotlin/tree/master/plugins
[marshalling]: https://en.wikipedia.org/wiki/Marshalling_(computer_science)
[power-assert]: https://groovy-lang.org/testing.html#_power_assertions
[power-assert-history]: https://dontmindthelanguage.wordpress.com/2009/12/11/groovy-1-7-power-assert/
[kotlin-power-assert]: https://github.com/bnorm/kotlin-power-assert
[kevin-most-kotlinconf-2018]: https://www.youtube.com/watch?v=w-GMlaziIyo
[ir-lowerings]: https://github.com/JetBrains/kotlin/tree/master/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower
[ir-lower-tailrec]: https://github.com/JetBrains/kotlin/blob/master/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower/TailrecLowering.kt
[ir-lower-concat]: https://github.com/JetBrains/kotlin/blob/master/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower/StringConcatenationLowering.kt
[ir-lower-suspend]: https://github.com/JetBrains/kotlin/blob/master/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower/AbstractSuspendFunctionsLowering.kt
[ir-lower-loop]: https://github.com/JetBrains/kotlin/blob/master/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower/loops/ForLoopsLowering.kt
[ir-builders]: https://github.com/JetBrains/kotlin/tree/master/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders
[short-circuiting]: https://en.wikipedia.org/wiki/Short-circuit_evaluation
[kotlin-compile-testing]: https://github.com/tschuchortdev/kotlin-compile-testing
