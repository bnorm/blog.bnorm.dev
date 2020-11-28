---
layout: post
title: Soft Assertion with kotlin-power-assert
description: Soft Assertion with kotlin-power-assert
tweet: https://twitter.com/bnormcodes/status/1310759111382966275?s=20
---

The [`kotlin-power-assert`][kotlin-power-assert] Kotlin compiler plugin is great (I may be biased as
its author) as it enables diagramming of assert function calls. As the plugin grows and matures I’m
discovering new use cases. One I wanted to try is the idea of soft assertions which [many]
[assertj-soft] [assertion][testng-soft] [libraries][testng-soft] [support][truth-expect]. Let’s take
a look quickly at an example of `verifyAll` from the [Spock Framework][spock].

```groovy
def "offered PC matches preferred configuration"() {
  when:
  def pc = shop.buyPc()

  then:
  verifyAll(pc) {
    vendor == "Sunny"
    clockRate >= 2333
    ram >= 406
    os == "Linux"
  }
}
```

The example above is straight from the documentation and demonstrates verifying multiple properties
of a class at a single time. The nice thing here is that all failures will be reported, not just the
first one. This avoids needing to run the test multiple times, discovering the next failed property
after fixing the previous failed property. The `verifyAll` function gathers each failed assertion and
only throws an exception when the scope closes.

# Complex Boolean Expressions

`kotlin-power-assert` allows complex boolean expressions to be diagramed but this requires support for
[short-circuiting]. This is because smart casting allows some interesting boolean expressions like the
following.

```kotlin
assert(jane != null && jane.firstName == "Jane")

# Example failure
java.lang.AssertionError: Assertion failed
assert(jane != null && jane.firstName == "Jane")
       |    |
       |    false
       null
```
       
So if multiple properties of a class are included as a single boolean expression, 
`kotlin-power-assert` will only diagram up to the first failure.

```kotlin
assert(jane.firstName == "Jane" && jane.lastName == "Doe")

# Example failure
assert(jane.firstName == "Jane" && jane.lastName == "Doe")
       |    |         |
       |    |         false
       |    John
       Person@765cb1b3
```

If the `Person` class had a better `toString()` function or was a data class that might solve this
specific problem, but this exposes a general problem that needs solving.

# Scoped Assertion

Creating something similar to Spock’s `verifyAll` with Kotlin and the `kotlin-power-assert` compiler
plugin is very simple and only takes a few dozen lines of code. Kotlin supports the same
higher-order function syntax as Groovy and `kotlin-power-assert` adds diagrammed assertion support.
Here’s all it takes:

```kotlin
typealias LazyMessage = () -> Any

interface AssertScope {
  fun assert(assertion: Boolean, lazyMessage: LazyMessage? = null)
}

private class SoftAssertScope : AssertScope {
  private val assertions = mutableListOf<Throwable>()

  override fun assert(assertion: Boolean, lazyMessage: LazyMessage?) {
    if (!assertion) {
      assertions.add(AssertionError(lazyMessage?.invoke()?.toString()))
    }
  }

  fun close(exception: Throwable? = null) {
    if (assertions.isNotEmpty()) {
      val base = exception ?: AssertionError("Multiple failed assertions")
      for (assertion in assertions) {
        base.addSuppressed(assertion)
      }
      throw base
    }
  }
}

fun <R> assertSoftly(block: AssertScope.() -> R): R {
  val scope = SoftAssertScope()
  val result = runCatching { scope.block() }
  scope.close(result.exceptionOrNull())
  return result.getOrThrow()
}
```

A [sample][assert-scope] of this was added to the `kotlin-power-assert` library in version 0.5.3
which fixed a bug when transforming member functions. Now you can write the following in your test!

```kotlin
val jane: Person = ...
assertSoftly {
  assert(jane.firstName == "Jane")
  assert(jane.lastName == "Doe")
}
```

---

Soft assertions can help pin point all failures with a single test run. `kotlin-power-assert` can
help diagram those failures and make them easier to triage. Give them both a try and see how they
work for you!

[kotlin-power-assert]: https://github.com/bnorm/kotlin-power-assert
[spock]: http://spockframework.org/

[assertj-soft]: https://joel-costigliola.github.io/assertj/core/api/org/assertj/core/api/SoftAssertions.html
[testng-soft]: https://www.javadoc.io/doc/org.testng/testng/6.8.8/org/testng/asserts/SoftAssert.html
[testng-soft]: http://spockframework.org/spock/docs/1.2-RC3/all_in_one.html#_using_code_verifyall_code_to_assert_multiple_expectations_together
[truth-expect]: https://truth.dev/api/latest/com/google/common/truth/Expect.html

[short-circuiting]: https://en.wikipedia.org/wiki/Short-circuit_evaluation
[assert-scope]: https://github.com/bnorm/kotlin-power-assert/blob/v0.5.3/sample/src/commonMain/kotlin/com/bnorm/power/AssertScope.kt
