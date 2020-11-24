---
layout: post
title: '"Unused" Type Parameters on Kotlin Inline Classes'
tweet: https://twitter.com/bnormcodes/status/1155875415740899329?s=20
---

Kotlin introduced inline classes in 1.3 as an experimental feature. They allow wrapping of a single
value which is inlined at compile time. You can use them for [ID classes] and also for
[logging implementations]. They are a great feature with some very interesting use cases.

"Unused" type parameters can actually be very useful on inline classes. Take the following as an
example.

```kotlin
@Suppress("unused")
inline class Key<T>(val value: String)
```

The above defines a Key inline class which has an associated type parameter and inlines to a String.
The Kotlin compiler claims this type parameter is unused but it can actually be used in function
signatures.

```kotlin
class Context(private val map: MutableMap<Key<*>, Any> = mutableMapOf()) {
  operator fun <T : Any> get(key: Key<T>): T? {
    return map.get(key)?.let { it as T }
  }

  operator fun <T : Any> set(key: Key<T>, value: T) {
    map.put(key, value)
  }
}
```

The above Context class is type safe at compile time but all the type parameter information is
erased at runtime. Create Keys for all the information you want to store in Context and getting and
setting will be type safe. [Feel free to play around with this example in a Kotlin Playground]
[playground].

```kotlin
data class Person(
  val id: Int,
  val name: String,
  val email: String
) {
  companion object {
    val idKey = Key<Int>("id")
    val nameKey = Key<String>("name")
    val emailKey = Key<String>("email")
  }
}

fun main() {
  val person = Person(123, "Brian", "brian@example.com")

  val context = Context()
  context[Person.idKey] = person.id
  context[Person.nameKey] = person.name
  context[Person.emailKey] = person.email

  //context[Person.idKey] = person.email // doesn't compile

  println(context[Person.emailKey])
}
```

I'm interested to see others' ideas on how this can be used so please share them in the comments!
Enjoy!

[ID classes]: https://jakewharton.com/inline-classes-make-great-database-ids
[logging implementations]: https://github.com/michaelbull/kotlin-inline-logger
[playground]: https://pl.kotl.in/oYL7nQPR6
