---
layout: post
title: Writing Your Second Kotlin Compiler Plugin, Part 3 — Navigating Kotlin IR
description: Navigating Kotlin IR
tweet: https://twitter.com/bnormcodes/status/1335261994769899522?s=20
---

> At the time of writing this article, [Kotlin compatibility] for IR backend is in Alpha status and
> the compiler plugin API is Experimental. As such, information contained in this article about IR
> and compiler plugins could be out-of-date or incorrect. If official documentation exists, please
> refer to it first.

Continuing on from [Part 2] where we learned how to inspect Kotlin IR, let's explore how to navigate
the Kotlin IR tree and inspect specific elements.

- [Part 1 - Project Setup][Part 1]
- [Part 2 - Inspecting Kotlin IR][Part 2]
- [Part 3 - Navigating Kotlin IR][Part 3]
- [Part 4 - Building Kotlin IR][Part 4]
- Part 5 - Transforming Kotlin IR
- Part 6 - ?
- Part ? - Publishing a Kotlin Compiler Plugin

# We Got Visitors!

As shown in [Part 2], the Kotlin IR is an [abstract syntax tree] (AST). This means that
fundamentally when dealing with Kotlin IR, we are dealing with a tree structure. We'll be dealing
with things like parents, siblings, and children which all implement the same interface,
[IrElement]. This interface shows us exactly how we will be navigating the Kotlin IR tree: with the
[visitor pattern]! The [IrElement] interface has 2 functions which enable the visitor pattern.

```kotlin
fun <R, D> accept(visitor: IrElementVisitor<R, D>, data: D): R
fun <D> acceptChildren(visitor: IrElementVisitor<Unit, D>, data: D): Unit
```

Both of these functions take an [IrElementVisitor], which is another interface but with a lot more
functions. An implementation of the [IrElementVisitor] interface is what allows us to navigate a
Kotlin IR tree, with the appropriate function being called when visiting that type of element.

# Top To Bottom

If we inspect the default implementations of the functions of [IrElementVisitor] we can learn a
little about how the different nodes of the Kotlin IR tree are related. For example, lets take a
look at the `visitClass` function.

```kotlin
fun visitClass(declaration: IrClass, data: D) = visitDeclaration(declaration, data)
```

This function accepts an [IrClass] and the default implementation delegates to another function,
`visitDeclaration`. If you want to perform some operation on every [IrClass], you would provide a
different implementation for this function. But let's first continue down the call chain and look at
the `visitDeclaration` function.

```kotlin
fun visitDeclaration(declaration: IrDeclarationBase, data: D) = visitElement(declaration, data)
```

This function accepts an [IrDeclarationBase] and the default implementation delegates to yet another
function, `visitElement`. Because we can call this function with an [IrClass], we know that IrClass
must implement [IrDeclarationBase]. Finally, the `visitElement` function.

```kotlin
fun visitElement(element: IrElement, data: D): R
```

This `visitElement` function is the only function which does not have a default implementation.
Since [IrElement] is the base interface for all IR elements, it makes sense that all functions in
[IrElementVisitor] eventually delegate to this `visitElement` function. This function is also what
helps enable recursion to children of each element!

# Parents & Siblings & Children, Oh My!

To further understand how the [IrElementVisitor] works, we can take a look at how [IrClass]
implements the 2 functions from [IrElement] which accept a visitor implementation.

```kotlin
override fun <R, D> accept(visitor: IrElementVisitor<R, D>, data: D): R =
    visitor.visitClass(this, data)

override fun <D> acceptChildren(visitor: IrElementVisitor<Unit, D>, data: D) {
    thisReceiver?.accept(visitor, data)
    typeParameters.forEach { it.accept(visitor, data) }
    declarations.forEach { it.accept(visitor, data) }
}
```

From this example, we can see that the `accept` function simply calls the appropriate visitor
function. This polymorphic behavior allows a caller to properly handle any [IrElement] simply by
calling `element.accept(visitor, data)`. This behavior of calling the appropriate visitor function
is consistent across all element implementations.

The `acceptChildren` calls the `accept` function on each child [IrElement] it contains. As we just
learned, the `accept` function simply calls the appropriate visitor function. This means that by
calling `element.acceptChildren(visitor, data)` the visitor will properly handle all children of
the element. This leads us to how recursion works with the visitor pattern!

Say we had the following visitor implementation.

```kotlin
class RecursiveVisitor : IrElementVisitor<Unit, Nothing?> {
  override fun visitElement(element: IrElement, data: Nothing?) {
    element.acceptChildren(this, data)
  }
}
```

By calling `moduleFragment.accept(RecursiveVisitor(), null)` from an [IrGeneratorExtension]
[Part 1 - Kotlin Plugin] we would visit every single [IrElement] in the module! Don't believe me?
Add some print statements and try it for yourself. Can you recreate a simplified version of the
[dump()] function we looked at in [Part 2]?

# Data In, Data Out

Something we skipped over with [IrElementVisitor] is the 2 type parameters it has, one defining the
type of the `data` parameter each visit function accepts, and the other defining the return type for
each visit function.

The input value `data` can be used to pass contextual information throughout the IR navigation. For
example, this could be the current indent spacing to use when printing out the details of an
element.

```kotlin
class StringIndentVisitor : IrElementVisitor<Unit, String> {
  override fun visitElement(element: IrElement, data: String) {
    println("$data${render(element)} {")
    element.acceptChildren(this, "  $data")
    println("$data}")
  }
}
```

The output type can be used to return a result from accepting the visitor. There are rare occasions
when this is useful (like IR transformation!) and usually the return type is just `Unit`. However,
one example might be to find the root parent of an element.

```kotlin
// Not as efficient as a while loop, but exemplifies how the output type could be used
class RootParentVisitor : IrElementVisitor<IrDeclarationParent?, Nothing?> {
  override fun visitElement(element: IrElement, data: Nothing?): IrDeclarationParent? = null

  override fun visitDeclaration(declaration: IrDeclarationBase, data: Nothing?): IrDeclarationParent {
    val parent = declaration.parent
    return parent.accept(this, null) ?: parent
  }
}
```

Something interesting to notice, is that the `acceptChildren` only accepts a visitor of type
`IrElementVisitor<Unit, D>`, which means that when performing recursive operations across all
children, no data can be returned. If data needs to be returned from a recursive visitor, it is
common pattern for the visitor to be passed some builder object which can be updated during
traversal.

```kotlin
class CollectingVisitor(
  private val elements: MutableList<IrElement>
) : IrElementVisitor<Unit, Nothing?> {
  override fun visitElement(element: IrElement, data: Nothing?) {
    elements.add(element)
    element.acceptChildren(this, data)
  }
}

fun collect(element: IrElement) = buildList<IrElement> {
  element.accept(CollectingVisitor(this), null)
}
```

# Depth Versus Breadth

The astute reader will have noticed at this point that the visitor pattern I have been explaining is
just a form of [tree traversal], and more specifically [depth-first traversal]. While it is possible
to perform a [breadth-first traversal], it is a bit more complicated. Like most breadth-first search
algorithms a queue of elements must be maintained and the children of the current element must be
added into that queue after processing the element.

```kotlin
fun breadthFirstCollect(element: IrElement) = buildList<IrElement> {
  val queue = ArrayDeque<IrElement>()
  val visitor = object : IrElementVisitor<Unit, Nothing?> {
    override fun visitElement(element: IrElement, data: Nothing?) {
      queue.add(element)
    }
  }

  queue.add(element)
  while (queue.isNotEmpty()) {
    val current = queue.removeFirst()
    this.add(current) // add element to collection
    current.acceptChildren(visitor, null) // add children to element queue
  }
}
```

So while breadth-first traversal is possible, I have not found any use for it when dealing with
Koltin IR, nor have I noticed any use of it in the Kotlin compiler code base.

---

Hopefully you now have a better understanding of how a Kotlin IR tree can be navigated. Remember, if
you want to experiment with IR navigation or build your own compiler plugin, check out my [GitHub
template] for IR based Kotlin compiler plugins!

[Kotlin compatibility]: https://kotlinlang.org/docs/reference/evolution/components-stability.html
[Part 1]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1
[Part 2]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-2
[Part 3]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-3
[Part 4]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-4
[Part 1 - Kotlin Plugin]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1#kotlin-plugin
[abstract syntax tree]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[IrElement]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/IrElement.kt
[visitor pattern]: https://en.wikipedia.org/wiki/Visitor_pattern
[IrElementVisitor]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/visitors/IrElementVisitor.kt#L23
[IrClass]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrClass.kt#L31
[IrDeclarationBase]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/declarations/IrDeclaration.kt#L48
[dump()]: https://github.com/JetBrains/kotlin/blob/1.4.20/compiler/ir/ir.tree/src/org/jetbrains/kotlin/ir/util/DumpIrTree.kt#L30
[tree traversal]: https://en.wikipedia.org/wiki/Tree_traversal
[depth-first traversal]: https://en.wikipedia.org/wiki/Depth-first_search
[breadth-first traversal]: https://en.wikipedia.org/wiki/Breadth-first_search
[GitHub template]: https://github.com/bnorm/kotlin-ir-plugin-template
