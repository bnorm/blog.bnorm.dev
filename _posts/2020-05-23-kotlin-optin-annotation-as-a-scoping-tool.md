---
layout: post
title: Kotlin OptIn Annotation as a Scoping Tool
tweet: https://twitter.com/bnormcodes/status/1264305096546025473?s=20
---

I like to follow the [`kotlinx.coroutines`] library closely on GitHub. Not really because I have
anything to contribute, but because there is so much to learn!

Take it’s use of the [`RequiresOptIn`] annotation (previously `Experimental` in Kotlin pre-1.3.70).
There are some really nice ways this annotation is used, especially when introducing new library
features like `Flow` and letting you know that parts of it are still experimental with the
[`FlowPreview`] annotation.

However, the one that sticks out to me is the [`InternalCoroutinesApi`] annotation. This annotation
marks APIs that have to be public but should not be used outside the library and will result in a
compiler error if the using code is not marked with `@OptIn(InternalCoroutinesApi::class)`. This is
used to **scope** those APIs!

How else can this be used to scope access? Let’s take inspiration from [using inline classes for
database IDs] and make an inline class to represent a password.

```kotlin
inline class Password(
  private val value: String
) {
  override fun toString(): String = "*****"
}
```

This gives us some protection when dealing with passwords, making sure you don’t pass a password to
a function not expecting one. But what about the function that needs to hash the password before
persisting? Or the function that needs to compare that password against an existing password hash?
Some things still need access to the raw password. Enter `RequiresOptIn`!

```kotlin
@RequiresOptIn(level = RequiresOptIn.Level.ERROR)
annotation class UnsafePassword

inline class Password(
  @property:UnsafePassword
  val value: String
) {
  override fun toString(): String = "*****"
}
```

Now any code that tries to access value will get a compiler error unless it uses
`@OptIn(UnsafePassword::class)`! The raw password has been scoped to only those functions that need
it, and made that scope even more explicit!

```kotlin
@OptIn(UnsafePassword::class)
fun hash(password: Password): String
  = // password.value is no longer an error
```

Bonus, you can now search for usages of UnsafePassword to find everywhere that is dealing with a raw
password.

> Disclaimer: I’m not a security expert. This password example is meant to be just that, an example.
> Make sure you are always following best practices when dealing with sensitive information.

[`kotlinx.coroutines`]: https://github.com/Kotlin/kotlinx.coroutines
[`RequiresOptIn`]: https://kotlinlang.org/docs/reference/opt-in-requirements.html
[`FlowPreview`]: https://github.com/Kotlin/kotlinx.coroutines/blob/1eeed509abbe8e542124841ce14884202463460b/kotlinx-coroutines-core/common/src/Annotations.kt#L40
[`InternalCoroutinesApi`]: https://github.com/Kotlin/kotlinx.coroutines/blob/1eeed509abbe8e542124841ce14884202463460b/kotlinx-coroutines-core/common/src/Annotations.kt#L66
[using inline classes for database IDs]: https://jakewharton.com/inline-classes-make-great-database-ids/
