---
layout: post
title: Writing Your Second Kotlin Compiler Plugin, Part 5 — Transforming Kotlin IR
description: Transforming Kotlin IR
tweet: https://twitter.com/bnormcodes/status/1340337338879229952?s=20
---

> At the time of writing this article, [Kotlin compatibility] for IR backend is in Alpha status and
> the compiler plugin API is Experimental. As such, information contained in this article about IR
> and compiler plugins could be out-of-date or incorrect. If official documentation exists, please
> refer to it first.

And here we are, ready to transform Kotlin IR! After learning how to navigate and build Kotlin IR we
now have the tools needed to transform Kotlin IR. This is a long one, so let's get going!

- [Part 1 - Project Setup][Part 1]
- [Part 2 - Inspecting Kotlin IR][Part 2]
- [Part 3 - Navigating Kotlin IR][Part 3]
- [Part 4 - Building Kotlin IR][Part 4]
- [Part 5 - Transforming Kotlin IR][Part 5]
- [Part 6 - Support Libraries, Publishing, and Integration Testing][Part 6]

# Déjà Vu

Just like in [Part 3] where we learned about the visitor pattern, transforming Kotlin IR is very
similar but with a specific sub-class of [IrElementVisitor] called [IrElementTransformer]. This
interface defines a return type of [IrElement] at the class level but then overrides each visit
function to be more specific types like [IrStatement] and [IrExpression]. And again, each
[IrElement] has 2 functions which take a transformer as the parameter.

```kotlin
fun <D> transform(transformer: IrElementTransformer<D>, data: D): IrElement = accept(transformer, data)
fun <D> transformChildren(transformer: IrElementTransformer<D>, data: D): Unit
```

The base `transform` function defaults to delegating the visitor function `accept` and overrides of
the function in sub-classes usually are only needed to override the return type of the function to a
more specific type. For example, the `transform` function in [IrFile] looks as follows.

```kotlin
override fun <D> transform(transformer: IrElementTransformer<D>, data: D): IrFile =
  accept(transformer, data) as IrFile
```

The `transformChildren` function is - once again - much more interesting. Just like visiting all the
children of each element, the transform function allows transforming each child element. For example
let's look at the implementation for [IrClass].

```kotlin
override fun <D> transformChildren(transformer: IrElementTransformer<D>, data: D) {
  thisReceiver = thisReceiver?.transform(transformer, data)
  typeParameters = typeParameters.transformIfNeeded(transformer, data)
  declarations.transformInPlace(transformer, data)
}
```

Since each child property is defined as a `var` or a mutable collection, if the transform function
returns a different instance, it can replace the previous instance. Also note that if a child
element does not exist, it will not be transformed so it cannot be replaced. However when
transforming an element, you can add child elements as needed which we will show later.

# Tracing The Outline

Inspired by [Kevin Most's 2018 KotlinConf talk][kevin-most-kotlinconf], which in turn was inspired
by [Jake Wharton's Hugo library][hugo], let's take the example of transforming functions to include
debug print statements. We'll just use `println` for logging but this could be expanded to make use
of a proper logging framework.

```kotlin
@DebugLog
fun greet(greeting: String = "Hello", name: String = "World"): String {
  return "${'$'}greeting, ${'$'}name!"
}
```

In our example, given a function annotated with `@DebugLog`, println statements will be added at the
entrance and all exits of the function. Here is what writing this by hand would look like.

```kotlin
@DebugLog
fun greet(greeting: String = "Hello", name: String = "World"): String {
  println("⇢ greet(greeting=$greeting, name=$name)")
  val startTime = TimeSource.Monotonic.markNow()
  try {
    val result = "${'$'}greeting, ${'$'}name!"
    println("⇠ greet [${startTime.elapsedNow()}] = $result")
    return result
  } catch (t: Throwable) {
    println("⇠ greet [${startTime.elapsedNow()}] = $t")
    throw t
  }
}
```

Yes, we are use 

# Autobots, Roll Out!

To perform this debug log transformation, we'll need a transformer implementation and ours will
extend from [IrElementTransformerVoidWithContext]. This abstract transformer takes no input data
("Void") and maintains an internal stack of various IR elements it has visited ("WithContext").

```kotlin
class DebugLogTransformer(
  private val pluginContext: IrPluginContext,
  private val annotationClass: IrClassSymbol,
  private val logFunction: IrSimpleFunctionSymbol,
) : IrElementTransformerVoidWithContext()
```

Our transformer will require 3 parameters:
- The current [IrPluginContext] for building IR elements,
- An [IrClassSymbol] for the annotation use to mark functions for debug logging,
- An [IrSimpleFunctionSymbol] for the function used to log debug messages.

We will also need a few local properties to reference known types, classes, and functions.

```kotlin
private val typeUnit = pluginContext.irBuiltIns.unitType
private val typeThrowable = pluginContext.irBuiltIns.throwableType

private val classMonotonic =
  pluginContext.referenceClass(FqName("kotlin.time.TimeSource.Monotonic"))!!

private val funMarkNow =
  pluginContext.referenceFunctions(FqName("kotlin.time.TimeSource.markNow"))
    .single()

private val funElapsedNow =
  pluginContext.referenceFunctions(FqName("kotlin.time.TimeMark.elapsedNow"))
    .single()
```

Next, we can override the `visitFunctionNew` function to intercept transformation of function
statements. This visit function is specific to [IrElementTransformerVoidWithContext] as [IrFunction]
is an element which is added to the context stack the abstract transformer maintains. As we know
from [Part 4][part-4-null-body], a function body can be null. So to see if we should transform an intercepted function
we need to check if it has a body and has the correct annotation applied. For the annotation check,
we can use the extension function `hasAnnotation`.

```kotlin
override fun visitFunctionNew(declaration: IrFunction): IrStatement {
  val body = declaration.body
  if (body != null && declaration.hasAnnotation(annotationClass)) {
    declaration.body = irDebug(declaration, body)
  }
  return super.visitFunctionNew(declaration)
}
```

Before tackling the `irDebug` function we need to create, let's first take a look at the IR helpers
for the entrance and exit log statements.

# Making An Entrance

When the function is first entered, we need to make a call to `println` to display the function name
and supplied parameter values.

```kotlin
println("⇢ greet(greeting=$greeting, name=$name)")
```

We created something similar to this in [Part 4][part-4-println] so the following should look
relatively familiar.

```kotlin
private fun IrBuilderWithScope.irDebugEnter(
  function: IrFunction
): IrCall {
  val concat = irConcat()
  concat.addArgument(irString("⇢ ${function.name}("))
  for ((index, valueParameter) in function.valueParameters.withIndex()) {
    if (index > 0) concat.addArgument(irString(", "))
    concat.addArgument(irString("${valueParameter.name}="))
    concat.addArgument(irGet(valueParameter))
  }
  concat.addArgument(irString(")"))

  return irCall(logFunction).also { call ->
    call.putValueArgument(0, concat)
  }
}
```

This does introduce a new builder function `irConcat`. [IrStringConcatenation] is a helpful IR
element for building interpolated strings and is used here to replicate Kotlin string templates but
at the IR level. Also notice the `irGet` builder which is able to access the value of the function
parameter.

# Making An Exit, Thrice!

On exit, we want to log the result or the exception thrown. If the function returns Unit we can skip
displaying the result as it is known to be nothing. This results in needing to replicate the
following 3 `println` statements.

```kotlin
println("⇠ greet [${startTime.elapsedNow()}] = $result")
println("⇠ greet [${startTime.elapsedNow()}] = $t")
println("⇠ greet [${startTime.elapsedNow()}]")
```

Similar to entrance logging, we can use an [IrStringConcatenation] again to build the message
string. However we need 2 additional pieces of information, the `startTime` and the `result` or
exception.

```kotlin
private fun IrBuilderWithScope.irDebugExit(
  function: IrFunction,
  startTime: IrValueDeclaration,
  result: IrExpression? = null
): IrCall {
  val concat = irConcat()
  concat.addArgument(irString("⇠ ${function.name} ["))
  concat.addArgument(irCall(funElapsedNow).also { call ->
    call.dispatchReceiver = irGet(startTime)
  })
  if (result != null) {
    concat.addArgument(irString("] = "))
    concat.addArgument(result)
  } else {
    concat.addArgument(irString("]"))
  }

  return irCall(logFunction).also { call ->
    call.putValueArgument(0, concat)
  }
}
```

The `startTime` is provided as an [IrValueDeclaration]. This is a reference to a local variable
which can be accessed using `irGet`. To use the `elapsedNow` function on the start time [TimeMark],
we can use the `funElapsedNow` symbol and provide the `startTime` variable as the `dispatchReceiver`
for the `irCall`. Note that the other receiver type of [IrCall], `extensionReceiver`, is used for
specifying the receiver for extension functions.

The `result` parameter is an [IrExpression] which will provide the result of the annotated function
or the exception thrown. This is optional for the case when the annotated function returns Unit. An
[IrExpression] can be added directly to an [IrStringConcatenation] as it can handle concatenating
arbitrary IR expressions.

# On Your Mark...

There are a couple other small section I want to go over before showing the full `irDebug` function
implementation.

To create a temporary local variable you can use the `irTemporary` builder. This in combination with
`irCall` and `irGetObject` we can save the start time for the annotated function. This is the same
as calling `TimeSource.Monotonic.markNow()` and saving to a local variable. Note that `irTemporary`
automatically adds the returned [IrVariable] to the surrounding [IrStatementsBuilder] so be careful
where you invoke this builder.

```kotlin
val startTime = irTemporary(irCall(funMarkNow).also { call ->
  call.dispatchReceiver = irGetObject(classMonotonic)
})
```

To build a try-catch statement, there unfortunately is no builder function so we need to construct
the implementation class directly. Since a `try` block is an expression in Kotlin, we need to
provide a result type for the [IrTry]. We also need to build a variable for each `catch` expression
to reference the caught throwable inside a `irCatch` builder. All of this will be built within a
[IrBuilderWithScope] which provides `scope`, `startOffset`, and `endOffset` as class properties.

```kotlin
val tryBlock: IrExpression = ...

val throwable = buildVariable(
  scope.getLocalDeclarationParent(), startOffset, endOffset, IrDeclarationOrigin.CATCH_PARAMETER,
  Name.identifier("t"), typeThrowable
)

IrTryImpl(startOffset, endOffset, tryBlock.type).also { irTry ->
  irTry.tryResult = tryResult
  irTry.catches += irCatch(throwable, ... as IrExpression)
}
```

The expression block of the `try` expression is built using `irBlock`. The original statements of
the annotated function body are added as statements to the `try` expression as well as a final call
to `irDebugExit` if the function returns Unit.

```kotlin
val tryBlock = irBlock(resultType = function.returnType) {
  for (statement in body.statements) +statement
  if (function.returnType == typeUnit) +irDebugExit(function, startTime)
}
```

The expression block of the `catch` expression is also built using `irBlock`. This block will use
`irDebugExit` to log the exception and then preserve the exception by rethrowing it.

```kotlin
irTry.catches += irCatch(throwable, irBlock {
  +irDebugExit(function, startTime, irGet(throwable))
  +irThrow(irGet(throwable))
})
```

# Get Set...

And here is the `irDebug` function, taking the annotated function and the original body as
parameters.

```kotlin
private fun irDebug(
  function: IrFunction,
  body: IrBody
): IrBlockBody {
  return DeclarationIrBuilder(pluginContext, function.symbol).irBlockBody {
    +irDebugEnter(function)

    val startTime = irTemporary(irCall(funMarkNow).also { call ->
      call.dispatchReceiver = irGetObject(classMonotonic)
    })

    val tryBlock = irBlock(resultType = function.returnType) {
      for (statement in body.statements) +statement
      if (function.returnType == typeUnit) +irDebugExit(function, startTime)
    }.transform(DebugLogReturnTransformer(function, startTime), null)

    val throwable = buildVariable(
      scope.getLocalDeclarationParent(), startOffset, endOffset, IrDeclarationOrigin.CATCH_PARAMETER,
      Name.identifier("t"), typeThrowable
    )

    +IrTryImpl(startOffset, endOffset, tryBlock.type).also { irTry ->
      irTry.tryResult = tryBlock
      irTry.catches += irCatch(throwable, irBlock {
        +irDebugExit(function, startTime, irGet(throwable))
        +irThrow(irGet(throwable))
      })
    }
  }
}
```

But, wait! What is this `DebugLogReturnTransformer` transformer on the `tryBlock`? This
secondary transformer is used to convert `return` statements so the result can be logged before
exiting the function.

```kotlin
val result = "${'$'}greeting, ${'$'}name!"
println("⇠ greet [${startTime.elapsedNow()}] = $result")
return result
```

To replicate the above transformation, we can intercept [IrReturn] elements with a secondary
transformer.

```kotlin
inner class DebugLogReturnTransformer(
  private val function: IrFunction,
  private val startTime: IrVariable
) : IrElementTransformerVoidWithContext() {
  override fun visitReturn(expression: IrReturn): IrExpression {
    if (expression.returnTargetSymbol != function.symbol) return super.visitReturn(expression)

    return DeclarationIrBuilder(pluginContext, function.symbol).irBlock {
      val result = irTemporary(expression.value)
      +irDebugExit(function, startTime, irGet(result))
      +expression.apply {
        value = irGet(result)
      }
    }
  }
}
```

We should only transform [IrReturn] expressions that return from the annotated function. This can be
checked by looking at the `returnTargetSymbol` property of the [IrReturn] element. To perform the
actual transformation we need to do 4 things:

1. Create a surrounding `irBlock` to hold multiple statements in a single expression,
2. Create a temporary variable using `irTemporary` to save the value returned,
3. Add our `irDebugExit` expression builder to log the result,
4. Add the original return to the block but instead return the value of the temporary variable.

# Go!

Now let's apply this debug log transformer against the provided `moduleFragment` of an
[IrGenerationExtension]. Grab a reference to the `DebugLog` annotation and `println` function from
the `pluginContext` and then call `transform` with an instance of `DebugLogTransformer` on
`moduleFragment`.

```kotlin
override fun generate(moduleFragment: IrModuleFragment, pluginContext: IrPluginContext) {
  val typeAnyNullable = pluginContext.irBuiltIns.anyNType

  val debugLogAnnotation = pluginContext.referenceClass(FqName("DebugLog"))!!
  val funPrintln = pluginContext.referenceFunctions(FqName("kotlin.io.println"))
    .single {
      val parameters = it.owner.valueParameters
      parameters.size == 1 && parameters[0].type == typeAnyNullable
    }

  moduleFragment.transform(DebugLogTransformer(pluginContext, debugLogAnnotation, funPrintln), null)
}
```

To see this in action, we can use the `kotlin-compile-testing` library as shown in [Part 1]
[part-1-testing]. With the compilation result object, we can get a [ClassLoader] of the compiled
classes. From this we can get the `main` function and run it.

```kotlin
val result = compile(
  sourceFile = SourceFile.kotlin(
    "main.kt",
    """
      annotation class DebugLog
      
      fun main() {
        println(greet())
        println(greet(name = "Kotlin IR"))
      }
      
      @DebugLog
      fun greet(greeting: String = "Hello", name: String = "World"): String { 
        Thread.sleep(15) // simulate work
        return "${'$'}greeting, ${'$'}name!"
      }
    """.trimIndent()
  )
)
assertEquals(KotlinCompilation.ExitCode.OK, result.exitCode)

val kClazz = result.classLoader.loadClass("MainKt")
val main = kClazz.declaredMethods.single { it.name == "main" && it.parameterCount == 0 }
main.invoke(null)
```

After running this test, we should see the following output.

```text
⇢ greet(greeting=Hello, name=World)
⇠ greet [20.1ms] = Hello, World!
Hello, World!
⇢ greet(greeting=Hello, name=Kotlin IR)
⇠ greet [15.6ms] = Hello, Kotlin IR!
```

---

I know this article was longer than usual and full of a lot of new things, but see if you can put
the pieces together and get something working! Can you make it so *all* functions in a *class* or
even *file* annotated with `@DebugLog` are transformed? (Hint:
`IrElementTransformerVoidWithContext.currentClass`) Can you make the `irDebugEnter` and
`irDebugExit` functions abstract so different logging frameworks and message formats could be used
by different implementations of `DebugLogTransformer`?

If you want to check out a working Kotlin compiler plugin created from all the code snippets above,
you can check out the [debuglog] repository. And if you want to try creating your own Kotlin
compiler plugin, take a look at the [GitHub template] repository I created. 

[Kotlin compatibility]: https://kotlinlang.org/docs/reference/evolution/components-stability.html
[kevin-most-kotlinconf]: https://www.youtube.com/watch?v=w-GMlaziIyo
[hugo]: https://github.com/JakeWharton/hugo
[Part 1]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1
[Part 2]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-2
[Part 3]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-3
[Part 4]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-4
[Part 5]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-5
[Part 6]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-6
[part-4-null-body]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-4#hello-world-but-make-it-ir
[part-4-println]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-4#hello-world-but-make-it-ir
[part-1-testing]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1#kotlin-plugin---testing
[IrElementVisitor]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/visitors/IrElementVisitor.kt#L23
[IrElementTransformer]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/visitors/IrElementTransformer.kt#L24
[IrElement]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/IrElement.kt#L23
[IrStatement]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/IrElement.kt#L37
[IrExpression]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrExpression.kt#L26
[IrFile]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFile.kt#L42
[IrClass]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrClass.kt#L31
[IrElementTransformerVoidWithContext]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/IrElementTransformerVoidWithContext.kt#L32
[IrPluginContext]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/extensions/IrPluginContext.kt#L20
[IrClassSymbol]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/symbols/IrSymbol.kt#L69
[IrSimpleFunctionSymbol]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/symbols/IrSymbol.kt#L99
[IrFunction]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFunction.kt#L31
[IrStringConcatenation]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrStringConcatenation.kt#L19
[IrValueDeclaration]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrValueDeclaration.kt#L13
[IrCall]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrCall.kt#L22
[IrVariable]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrVariable.kt#L24
[IrStatementsBuilder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/IrBuilder.kt#L44
[IrTry]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrTry.kt#L23
[IrBuilderWithScope]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/IrBuilder.kt#L37
[IrReturn]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrReturn.kt#L21
[IrGenerationExtension]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/extensions/IrGenerationExtension.kt#L12
[TimeMark]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.time/-time-mark/
[ClassLoader]: https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/lang/ClassLoader.html
[GitHub template]: https://github.com/bnorm/kotlin-ir-plugin-template
[debuglog]: https://github.com/bnorm/debuglog
