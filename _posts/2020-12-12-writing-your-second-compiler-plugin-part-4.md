---
layout: post
title: Writing Your Second Kotlin Compiler Plugin, Part 4 — Building Kotlin IR
description: Building Kotlin IR
tweet: https://twitter.com/bnormcodes/status/1337824700270080008?s=20
---

> At the time of writing this article, [Kotlin compatibility] for IR backend is in Alpha status and
> the compiler plugin API is Experimental. As such, information contained in this article about IR
> and compiler plugins could be out-of-date or incorrect. If official documentation exists, please
> refer to it first.

Now that we know how to navigate Kotlin IR from [Part 3], we can start talking about how to
transform Kotlin IR, right? Well we have one more topic we need to cover first: building Kotlin IR.
Transforming Kotlin IR works by adding, removing, or replacing IR elements, so we need to know how
to build them first!

- [Part 1 - Project Setup][Part 1]
- [Part 2 - Inspecting Kotlin IR][Part 2]
- [Part 3 - Navigating Kotlin IR][Part 3]
- [Part 4 - Building Kotlin IR][Part 4]
- Part 5 - Transforming Kotlin IR
- Part 6 - ?
- Part ? - Publishing a Kotlin Compiler Plugin

# Building Context

Back in [Part 1] we learned about the [IrGenerationExtension] interface which is the entry point for
a compiler plugin to generate or transform Kotlin IR. We talked briefly about the `IrModuleFragment`
parameter in [Part 2], but now we need to talk about the other parameter, `IrPluginContext`. A
[IrPluginContext] provides contextual information to the plugin about things outside the current
module being compiled.

From the [IrPluginContext] instance, we can obtain an instance of [IrFactory]. This factory class is
how a Kotlin compiler plugin can create its own IR elements. It contains many functions for building
instances of [IrClass][irclass-builder], [IrSimpleFunction][irfun-builder],
[IrProperty][irprop-builder], and much more. But probably more important, there are also quite a few
[extension functions][declaration-builders] which make building IR declarations easier.

[IrFactory] is useful when building declarations: classes, functions, properties, etc, but when
building statements and expressions, you will need an instance of [IrBuilder]. More importantly, you
will need an instance of [IrBuilderWithScope]. With this builder instance, a lot more [extension
functions][expression-builders] for IR expressions become available.

# Hello World, But Make It IR

Some of this might make more sense when shown an example, so let's build the following with IR!

```kotlin
fun main() {
  println("Hello, World!")
}
```

But how do we know what to build? Remember our friend [`dump()` from Part 2][Part 2-dump]? Using
that utility we can dump the IR tree for the above example.

```text
FUN name:main visibility:public modality:FINAL <> () returnType:kotlin.Unit
  BLOCK_BODY
    CALL 'public final fun println (message: kotlin.Any?): kotlin.Unit [inline] declared in kotlin.io.ConsoleKt' type=kotlin.Unit origin=null
      message: CONST String type=kotlin.String value="Hello, World!"
```

From this we have a blueprint for what to build! Let's take a look at different sections of the 
resulting code and then put the whole thing together.

```kotlin
val typeNullableAny = pluginContext.irBuiltIns.anyNType
val typeUnit = pluginContext.irBuiltIns.unitType
```

From the [IrPluginContext] you can get an instance of [IrBuiltIns] which provides quick access to
many things built into the Kotlin language. This includes references to many class types and IR
symbols. Here we are getting the types `Any?` and `Unit` as instances of [IrType]. These will be
used later to find the correct `println` overload and set the return type of the `main` function.

```kotlin
val funPrintln = pluginContext.referenceFunctions(FqName("kotlin.io.println"))
  .single {
    val parameters = it.owner.valueParameters
    parameters.size == 1 && parameters[0].type == typeNullableAny
  }
```

If what you need isn't built into the language itself, but rather comes from a dependency (like the
standard library), you can use the `IrPluginContext.reference*()` functions to find the needed
[IrSymbol]. An [IrSymbol] allows for a lazy reference to an [IrElement], and as such it has a
property `owner` which links to the referenced [IrElement]. Here we are getting the
[IrFunctionSymbol] for the `println(message: Any?)` function.

Since function overloading is supported in Kotlin, there can be multiple functions with the same
fully-qualified name, so we need to filter the returned functions to a single function with the
desired signature.

```kotlin
val funMain = pluginContext.irFactory.buildFun {
  name = Name.identifier("main")
  visibility = DescriptorVisibilities.PUBLIC // default
  modality = Modality.FINAL // default
  returnType = typeUnit
}
```

Using the [IrFactory] instance from the [IrPluginContext] we can build a function using the
extension function `buildFun`. This function takes a lambda with receiver of type
[IrFunctionBuilder] which allows for setting various properties like name, visibility, modality, and
the return type, all of which we are setting here even though some have default values.

You do not need to set a body as part of building the initial function definition. This is because
not all functions have bodies! Kotlin has many cases when a function body isn't required like
`abstract` and `expect` function definitions.

```kotlin
funMain.body = DeclarationIrBuilder(pluginContext, funMain.symbol).irBlockBody {
  val callPrintln = irCall(funPrintln)
  callPrintln.putValueArgument(0, irString("Hello, World!"))
  +callPrintln
}
```

If you do want to add a body to the function you can do so by setting the `body` property of the
built [IrFunction]. To do so you will need an instance of [IrBuilderWithScope] which is created here
as a [DeclarationIrBuilder]. From this builder you will have access to [many builder extension
functions][expression-builders] including the `irBlockBody` function used here. The `irBlockBody`
takes a lambda with receiver of type [IrBlockBodyBuilder] which is also a [IrBuilderWithScope]
allowing other builder extension functions to be used.

From within an [IrBlockBodyBuilder], a function call can be created via the `irCall` function and a
string constant can be created via the `irString` function. Combine these together to create a call
to `println` with the parameter value `"Hello, World!"`. This function call can be added to the
surrounding block body using the `+` operator on the [IrCall]. Otherwise we would have created the
function call but not put it anywhere!

Putting all these sections together, and using the `also` scoping function to remove temporary local
variables, will result in the following. This example should print out a dump of the built IR which
is exactly the same as what we started with; success!

```kotlin
override fun generate(moduleFragment: IrModuleFragment, pluginContext: IrPluginContext) {
  val typeNullableAny = pluginContext.irBuiltIns.anyNType
  val typeUnit = pluginContext.irBuiltIns.unitType

  val funPrintln = pluginContext.referenceFunctions(FqName("kotlin.io.println"))
    .single {
      val parameters = it.owner.valueParameters
      parameters.size == 1 && parameters[0].type == typeNullableAny
    }

  val funMain = pluginContext.irFactory.buildFun {
    name = Name.identifier("main")
    visibility = DescriptorVisibilities.PUBLIC // default
    modality = Modality.FINAL // default
    returnType = typeUnit
  }.also { function ->
    function.body = DeclarationIrBuilder(pluginContext, function.symbol).irBlockBody {
      +irCall(funPrintln).also { call ->
        call.putValueArgument(0, irString("Hello, World!"))
      }
    }
  }

  println(funMain.dump())
}
```

# Origin Story

The pattern of dumping Kotlin IR and recreating is the pattern I most often use when attempting to
create IR elements. However, sometimes it doesn't work so well. For example, consider the following
data class.

```kotlin
data class Person(
  val name: String,
  val age: Int,
  val email: String
)
```

If you dump the IR tree for this class it is 181 lines long! Because of all the code generation that
happens behind a data class, it becomes difficult to manually generate efficiently. All of this
generation takes place during the conversion between the compiler frontend (PSI/FIR) to the compiler
backend (IR). However, it does hint at something which can be useful so let's take a look at a small
section of the IR dump.

```text
FUN GENERATED_DATA_CLASS_MEMBER name:component1 visibility:public modality:FINAL <> ($this:<root>.Person) returnType:kotlin.String [operator]
  $this: VALUE_PARAMETER name:<this> type:<root>.Person
  BLOCK_BODY
    RETURN type=kotlin.Nothing from='public final fun component1 (): kotlin.String [operator] declared in <root>.Person'
      GET_FIELD 'FIELD PROPERTY_BACKING_FIELD name:name type:kotlin.String visibility:private [final]' type=kotlin.String origin=null
        receiver: GET_VAR '<this>: <root>.Person declared in <root>.Person.component1' type=<root>.Person origin=null
```

This is the IR for the generated `fun component1(): String` function of the data class which will
return the name of the person. Notice the big `GENERATED_DATA_CLASS_MEMBER` text right near the
beginning of the dump? This is the [IrDeclarationOrigin] property of the function that all
[IrDeclaration] elements have. The origin of a declaration describes what Kotlin source code
construct compilation resulted in that particular declaration. In our example, this function
originated from a data class.

There is also a [IrStatementOrigin] class which is the same for many different subtypes of
[IrStatement] which includes the likes of `DO_WHILE_LOOP`, `ELVIS`, and `RANGE`. From these types,
and if you have the Kotlin source code opened with IntelliJ, you can navigate to the code which
generates the IR code for corresponding construct. For example, if you search for `DO_WHILE_LOOP`,
you will find the [LoopExpressionGenerator] class which has a function called `generateDoWhileLoop`
which can generate a do-while loop `IrExpression` from a `KtDoWhileExpression`.

If you are attempting to build an IR element and do not know what functions to call to build it,
often the easiest thing is to search the Kotlin compiler for examples of similar IR elements. Having
a local instance of the Kotlin repository which has been indexed by IntelliJ is very import since
there is no official documentation for a compiler plugin API.

---

Hopefully this article provided you with a good foundation for building IR elements. If you are
struggling with the concepts of code generation in general, you might want to check out something
like [KotlinPoet] which generates Kotlin source code instead of IR and is extremely well documented.
Since [KotlinPoet] generates text output it is also easier to experiment with different However, if
you want to jump feet first into building some Kotlin IR elements, check out my [GitHub template]
project for IR based Kotlin compile plugins!

[Kotlin compatibility]: https://kotlinlang.org/docs/reference/evolution/components-stability.html
[Part 1]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1
[Part 2]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-2
[Part 3]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-3
[Part 4]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-4
[Part 2-dump]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-2
[IrGenerationExtension]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/extensions/IrGenerationExtension.kt
[IrPluginContext]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/extensions/IrPluginContext.kt
[IrFactory]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFactory.kt
[irclass-builder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFactory.kt#L28
[irfun-builder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFactory.kt#L89
[irprop-builder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFactory.kt#L136
[declaration-builders]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/ir/builders/declarations/declarationBuilders.kt
[IrBuilder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/IrBuilder.kt#L31
[IrBuilderWithScope]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/IrBuilder.kt#L37
[IrCall]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrCall.kt#L22
[expression-builders]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/ExpressionHelpers.kt
[IrBuiltIns]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/descriptors/IrBuiltIns.kt
[IrType]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/types/IrType.kt
[IrSymbol]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/symbols/IrSymbol.kt#L27
[IrElement]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/IrElement.kt#L23
[IrFunctionSymbol]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/symbols/IrSymbol.kt#L93
[IrFunctionBuilder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/declarations/IrFunctionBuilder.kt
[IrFunction]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrFunction.kt#L31
[DeclarationIrBuilder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower/LowerUtils.kt#L42
[IrBlockBodyBuilder]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/builders/IrBuilder.kt#L58
[IrDeclarationOrigin]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrDeclarationOrigin.kt#L19
[IrDeclaration]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrDeclaration.kt#L37
[IrStatementOrigin]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/expressions/IrStatementOrigin.kt#L23
[IrStatement]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/IrElement.kt#L37
[LoopExpressionGenerator]: https://github.com/JetBrains/kotlin/blob/v1.4.20/compiler/ir/ir.psi2ir/src/org/jetbrains/kotlin/psi2ir/generators/LoopExpressionGenerator.kt#L55
[KotlinPoet]: https://square.github.io/kotlinpoet/
[GitHub template]: https://github.com/bnorm/kotlin-ir-plugin-template
