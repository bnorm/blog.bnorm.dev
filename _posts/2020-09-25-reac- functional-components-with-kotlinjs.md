---
layout: post
title: React Functional Components with Kotlin/JS
description: React Functional Components with Kotlin/JS
---

I’ve been playing with Kotlin/JS and React lately. I’ve been wanting to learn more about frontend
development, and since I’m very familiar with Kotlin, thought Kotlin/JS and React would be a good
entry point.

# React

I’m not even going to pretend to be able to explain React. You should go read the [official 
documentation] if you want to learn more. A quick primer: React enables the creation of
**Components** which are reusable and composable and built from other components and standard HTML
tags using a XML-like syntax called JSX. Components can be stateful and take properties as inputs.
Here’s an example component right from the home page of React:

```JavaScript
class HelloMessage extends React.Component {
  render() {
    return (
      <div>
        Hello, {this.props.name}
      </div>
    );
  }
}
```

# React Functional Components

More modern React uses [**`Functional Components`**]. These are just standard JavaScript functions
which accept a `props` parameter and return a React element. Here is the functional version of the
previous component example.

```JavaScript
function Welcome(props) {
  return <div>Hello, {props.name}</div>;
}
```

# Kotlin/JS and React

Using the [wrappers for React] provided by JetBrains, you can define React components in Kotlin/JS.
Because Kotlin is a type safe language, there is a bit more boiler plate required to reproduce the
original example.

```kotlin
external interface WelcomeProps : RProps {
    var name: String
}

class Welcome: RComponent<WelcomeProps, RState>() {
     override fun RBuilder.render() {
        div {
            +"Hello, ${props.name}"
        }
    }
}

fun RBuilder.welcome(name: String) = child(Welcome::class) {
    attrs.name = name
}
```

# Kotlin/JS and React Functional Components

Also provided by the Kotlin wrappers for React is a functional component builder. Again, because of
type safety, there is some builder plate and it is both similar and different from the class version
of the component.

```kotlin
external interface WelcomeProps : RProps {
    var name: String
}

val WELCOME = rFunction<WelcomeProps>("Welcome") { props -> 
    div {
        +"Hello, ${props.name}"
    }
}

fun RBuilder.Welcome(name: String) = WELCOME.invoke {
    attrs.name = name
}
```

# Kotlin/JS React Compiler Plugin

I’ve got some background with [Kotlin compiler plugins], so I looked at the boiler plate required to
build functional components in Kotlin/JS and knew it could be better. The most optimal reduction of
boiler plate might look something like the following.

```kotlin
fun RBuilder.Welcome(name: String) {
    div {
        +"Hello, $name"
    }
}
```

How do we get this function to be a valid Kotlin/JS React functional component? Some things to
notice:

1. The parameters to the function are exactly the properties of the `WelcomeProps` interface.

2. The body of the function is exactly the lambda used for the call to `rFunction`.

# Kotlin React Function Compiler Plugin

With the new Kotlin/JS compiler plugin [kotlin-react-function] for React functional components, the
above function can be converted using a single annotation `@RFunction`.

```kotlin
@RFunction
fun RBuilder.Welcome(name: String) {
    div {
        +"Hello, $name"
    }
}
```

When the annotation is added to the function along with the `kotlin-react-function` Gradle plugin,
the boiler plate required is automatically generated and the function is transformed to use the
boiler plate.

Looking back at the JavaScript version of this functional component, this is very similar and
reduces the boilerplate required significantly.

---

My Kotlin/JS and React journey is just beginning, but hopefully this compiler plugin will make
creating React functional components in Kotlin/JS much easier for me and everyone else!

[offical documentation]: https://reactjs.org/
[**`Functional Components`**]: https://reactjs.org/docs/components-and-props.html#function-and-class-components
[wrappers for React]: https://github.com/JetBrains/kotlin-wrappers
[Kotlin compiler plugins]: https://medium.com/@bnorm/exploring-kotlin-ir-bed8df167c23
[kotlin-react-function]: https://github.com/bnorm/kotlin-react-function
