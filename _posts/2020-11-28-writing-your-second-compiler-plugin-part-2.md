---
layout: post
title: Writing Your Second Kotlin Compiler Plugin, Part 2 — Inspecting Kotlin IR
description: Inspecting Kotlin IR
tweet: https://twitter.com/bnormcodes/status/1332744643633623041?s=20
---

> At the time of writing this article, [Kotlin compatibility] for IR backend is in Alpha status and
> the compiler plugin API is Experimental. As such, information contained in this article about IR
> and compiler plugins could be out-of-date or incorrect. If official documentation exists, please
> refer to it first.

In [Part 1] we learned how to set up a Gradle project for building a Kotlin compiler plugin. In this
part we will drive right into the structure of Kotlin IR and what it looks like.

- [Part 1 - Project Setup][Part 1]
- [Part 2 - Inspecting Kotlin IR][Part 2]
- [Part 3 - Navigating Kotlin IR][Part 3]
- Part 4 - Building Kotlin IR
- Part 5 - Transforming Kotlin IR
- Part 6 - ?
- Part ? - Publishing a Kotlin Compiler Plugin

# Foundation

Kotlin intermediate representation (IR) is an [abstract syntax tree] for representing Kotlin code.
Kotlin code is represented this way because Kotlin is a high-level programming language, allowing
for many advanced programming concepts. Before it can be converted to JVM byte code, JavaScript, or
LLVM's own IR, these high-level concepts need to be lowered. Many of these "lowerings" are the same 
across all Kotlin platforms - `suspend` functions for example - which makes having a common backend
representation for Kotlin extremely useful. (Not to mention useful for compiler plugins!)

Every node in the Kotlin IR syntax tree implements [IrElement]. Elements of the syntax tree
represent things like modules, packages, files, classes, properties, functions, parameters, `if`
statements, function invocations, and much, much, more. Let's see what this actually looks like!

# Your New Best Friend: `IrElement.dump()`

From [Part 1] we learned our entry point into IR is through a `IrGenerationExtension` instance. This
class must implement a function which takes a `IrModuleFragment` and `IrPluginContext` parameter.
Given a compiler plugin with the following implementation:

```kotlin
override fun generate(moduleFragment: IrModuleFragment, pluginContext: IrPluginContext) {
  println(moduleFragment.dump())
}
```

And a simple program with the following code:

```kotlin
fun main() {
  println(debug())
}

fun debug(name: String = "World") = "Hello, $name!"
```

The compiler plugin will output the following:

```text
MODULE_FRAGMENT name:<main>
  FILE fqName:<root> fileName:/var/folders/dk/9hdq9xms3tv916dk90l98c01p961px/T/Kotlin-Compilation3223338783072974845/sources/main.kt
    FUN name:main visibility:public modality:FINAL <> () returnType:kotlin.Unit
      BLOCK_BODY
        CALL 'public final fun println (message: kotlin.Any?): kotlin.Unit [inline] declared in kotlin.io.ConsoleKt' type=kotlin.Unit origin=null
          message: CALL 'public final fun debug (name: kotlin.String): kotlin.String declared in <root>' type=kotlin.String origin=null
    FUN name:debug visibility:public modality:FINAL <> (name:kotlin.String) returnType:kotlin.String
      VALUE_PARAMETER name:name index:0 type:kotlin.String
        EXPRESSION_BODY
          CONST String type=kotlin.String value="World"
      BLOCK_BODY
        RETURN type=kotlin.Nothing from='public final fun debug (name: kotlin.String): kotlin.String declared in <root>'
          STRING_CONCATENATION type=kotlin.String
            CONST String type=kotlin.String value="Hello, "
            GET_VAR 'name: kotlin.String declared in <root>.debug' type=kotlin.String origin=null
            CONST String type=kotlin.String value="!"
```

The extension function `IrElement.dump()` has been my favorite tool when developing IR compiler
plugins. It allows for dumping an IR tree from any specific point and saving the output from run to
run. This has allowed me to write the code I want generated and take a dump of it for comparing
against as I build the IR compiler plugin generation. It has been my best friend.

Let's break this IR dump down into manageable pieces and compare it against the simple program.

# The Pieces

## `fun main()`

```text
FUN name:main visibility:public modality:FINAL <> () returnType:kotlin.Unit
```

This is the `main()` function definition from our simple program above. It defines the function with
the name, visibility, modality, and type signature of the function. We can clearly see that this is
a `public` and `final` function named `main` which takes no parameters - value (`()`) or type
(`<>`) - and returns `kotlin.Unit`.

## `fun debug(name: String = ...)`

```text
FUN name:debug visibility:public modality:FINAL <> (name:kotlin.String) returnType:kotlin.String
```

Vary similar to `main()`, but the function `debug()` takes a single value parameter named `name`
which is of type `String` and also returns a `String`. Notice the default value is not included in
the function definition dump but rather part of the parameter dump. Functions with and without
default parameters have the same signature, they are just called differently as we will see later.

## `name: String = "World"`

```text
VALUE_PARAMETER name:name index:0 type:kotlin.String
  EXPRESSION_BODY
    CONST String type=kotlin.String value="World"
```

The value parameter `name` for the function `debug()` is defined as having an `EXPRESSION_BODY`
which is just the constant string `World`. In Kotlin IR, there are mainly 2 different types of
"bodies": `BLOCK_BODY` and `EXPRESSION_BODY`. An expression body holds a single expression while a
block body can hold multiple statements. [Kotlin Academy has a great article]
[statement-vs-expression] on the difference between statements and expressions but to simplify: a
statement is a line of code and an expression is a statement which returns a value. The constant
string `World` is our first example of an expression.

## `"Hello, $name!"`

```text
FUN name:debug visibility:public modality:FINAL <> (name:kotlin.String) returnType:kotlin.String
  ...
  BLOCK_BODY
    RETURN type=kotlin.Nothing from='public final fun debug (name: kotlin.String): kotlin.String declared in <root>'
      STRING_CONCATENATION type=kotlin.String
        CONST String type=kotlin.String value="Hello, "
        GET_VAR 'name: kotlin.String declared in <root>.debug' type=kotlin.String origin=null
        CONST String type=kotlin.String value="!"
```

The body of the `debug()` function is a `BLOCK_BODY` with a single statement, a `return`. According
to the IR dump, this return has a type of `Nothing` which makes sense if you remember that [`return`
in Kotlin is actually an *expression*][return-expression]. The value being returned is also an
expression, in this case a string, which is the concatenation of 3 expressions: `"Hello, "`, getting
the value of the `name` parameter, and `"!"`.

## `println(debug())`

```text
CALL 'public final fun println (message: kotlin.Any?): kotlin.Unit [inline] declared in kotlin.io.ConsoleKt' type=kotlin.Unit origin=null
  message: CALL 'public final fun debug (name: kotlin.String): kotlin.String declared in <root>' type=kotlin.String origin=null
```

Function calls include which function is being called along with any parameters being provided. In
this example, the `println()` call includes a single parameter named `message`. This parameter value
is an expression which is another function call to `debug()`. Notice that no parameter values are
defined for the `debug()` function even though it has a single parameter `name`. This is because
`name` has a default value and therefore specifying a value is not required.

Also note that the `println()` function is an `inline` function. [Inlining is a "lowering"]
[inline-lowering] which is performed later in the compilation. This allows IR plugins to generate
calls to inline functions without having to perform the inlining.

# Dump The World

Try dumping some simple programs of your own! The [GitHub template] for Kotlin IR compiler plugins
is ready for you to edit the unit tests and compiler plugin! What code changes cause the IR dump to
change? What codes changes *don't* cause the IR dump to change? What does the dump of an `if`
statement in Kotlin IR look like? (hint: how are `when` and `if` similar?)

[Kotlin compatibility]: https://kotlinlang.org/docs/reference/evolution/components-stability.html
[Part 1]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1
[Part 2]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-2
[Part 3]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-3
[abstract syntax tree]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[IrElement]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/IrElement.kt
[statement-vs-expression]: https://blog.kotlin-academy.com/kotlin-programmer-dictionary-statement-vs-expression-e6743ba1aaa0
[return-expression]: https://kotlinlang.org/docs/reference/returns.html
[inline-lowering]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/backend.common/src/org/jetbrains/kotlin/backend/common/lower/inline/FunctionInlining.kt
[GitHub template]: https://github.com/bnorm/kotlin-ir-plugin-template
