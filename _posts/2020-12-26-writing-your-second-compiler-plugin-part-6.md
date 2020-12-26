---
layout: post
title: Writing Your Second Kotlin Compiler Plugin, Part 6 — Support Libraries, Publishing, and Integration Testing
description: Support Libraries, Publishing, and Integration Testing
---

> At the time of writing this article, [Kotlin compatibility] for IR backend is in Alpha status and
> the compiler plugin API is Experimental. As such, information contained in this article about IR
> and compiler plugins could be out-of-date or incorrect. If official documentation exists, please
> refer to it first.

As I wrap up this series of articles, there are a couple additional topics I wanted to cover in
regards to Kotlin compiler plugins. None of these were worth a dedicated article, as they are each
relatively small topics. Namely the ideas of support libraries, integration testing, and publishing
a compiler plugin.

- [Part 1 - Project Setup][Part 1]
- [Part 2 - Inspecting Kotlin IR][Part 2]
- [Part 3 - Navigating Kotlin IR][Part 3]
- [Part 4 - Building Kotlin IR][Part 4]
- [Part 5 - Transforming Kotlin IR][Part 5]
- [Part 6 - Support Libraries, Publishing, and Integration Testing][Part 6]

# Support Libraries

A support library is required for a Kotlin compiler plugin if the plugin expects certain symbols to
be present at compile time. In [Part 5][part-5-transformer], the transformer created expected an
annotation `DebugLog` to be present. In practice this annotation would be provided by a support
library which the Gradle plugin can automatically add to the dependencies of the project.

A support library is just another Kotlin project and is set up in the same way. The interesting part
happens when you want to automatically include this project as a dependency when your compiler
plugin is applied to a project. To do this, we'll need to go all the way back to
[Part 1][part-1-gradle-plugin] when we configured the Gradle `KotlinCompilerPluginSupportPlugin`
class. Using the `applyToCompilation` override function we can add a dependency to the project using
the `kotlinCompilation` parameter.

```kotlin
override fun applyToCompilation(
  kotlinCompilation: KotlinCompilation<*>
): Provider<List<SubpluginOption>> {
  kotlinCompilation.dependencies {
    implementation("${BuildConfig.ANNOTATION_LIBRARY_GROUP}:${BuildConfig.ANNOTATION_LIBRARY_NAME}:${BuildConfig.ANNOTATION_LIBRARY_VERSION}")
  }
  
  ...
}
```

Here we are adding the annotation library to the `implementation` dependency configuration. The
coordinates of the annotation library are automatically generated with the [buildconfig] Gradle
plugin.

```kotlin
buildConfig {
  ...
  
  val annotationProject = project(":debuglog-annotation")
  buildConfigField("String", "ANNOTATION_LIBRARY_GROUP", "\"${annotationProject.group}\"")
  buildConfigField("String", "ANNOTATION_LIBRARY_NAME", "\"${annotationProject.name}\"")
  buildConfigField("String", "ANNOTATION_LIBRARY_VERSION", "\"${annotationProject.version}\"")
}
```

By adding this configuration to the Gradle plugin, the user of your compiler plugin does not need to
worry about explicitly adding the support library to the list of dependencies.

> There is an open issue against Kotlin, [KT-43385], for an issue with IntelliJ and AndroidStudio
> not properly resolving dependencies added in this way. Gradle successfully builds the project, but
> the IDE will render the code with errors.

# Publishing

I'll keep this section short because I claim no expertise on this subject and there are many better
guides available depending on what you are trying to achieve. However, I wanted to point out a
couple things I have found helpful which will hopefully get you on the right path.

To publish the Gradle plugin, you can use the officially supported [`plugin-publish` Gradle plugin
from the Gradle team][plugin-publish]. This will require an account with Gradle and [there is a list
of steps to follow available][plugin-publish-steps].

To publish all the Kotlin projects (plugin, native plugin, and optional support library), you will
need to use a Maven repository host. Two common options are [Bintray by JFrog][bintray] and [Nexus
by Sonatype][nexus]. Nexus is what I use to sync with MavenCentral but [there are many steps to
complete][nexus-setup] before you are allowed to publish. Bintray can be much easier to upload with
the requirement of specifying additional Gradle repositories on the consumption side.

Publishing is a tricky topic and there are a lot of different ways to get your plugin in other
people's hands. I find it easiest to explore publication Gradle code in other projects so I will
link a few which I use as reference. Many take advantage of the [gradle-maven-publish-plugin] plugin
to perform publishing but I have no personal experience with it.

- [exhaustive by Square][exhaustive]
- [redacted-compiler-plugin by Zac Sweers][redacted-compiler-plugin]
- [kotlin-power-assert by Brian Norman][kotlin-power-assert] (yes, I constantly reference my own)

# Integration Testing

If you want to test local changes of your Kotlin compiler plugin, there is an extremely handy
feature in Gradle called [composite builds]. This is very similar to multi-project builds, instead
of doing it at the dependency level, you configure it at the root of the project. The documentation
for this feature is excellent, and while the documentation makes it obvious what the benefits are
for project level dependencies, it doesn't mention that dependency substitution also works at the
build dependency level (ie, Gradle plugins).

Composite builds allow for testing your Kotlin compiler plugin against a local project without the
need of publishing to a remote or local repository. If you include the build of your Kotlin compiler
plugin project in another Gradle project, the Gradle plugin and all other dependencies will be
substituted with the local compiler plugin project artifacts.

To illustrate this, create a directory in you Kotlin compiler plugin project called `sample`. Fill
this directory with a Gradle project, wrapper and all, that uses the Kotlin compiler plugin. In the
`settings.gradle` or `settings.gradle.kts` files you can add `includeBuild("..")` and when you build
the project in the `sample` directory, it will automatically build and use the Kotlin compiler
plugin in the parent directory. You can see an example of this at work in the
[debuglog][debuglog-integration] repository.

This is extremely useful for all kinds of testing, especially when you want to perform integration
testing against all of Kotlin's platforms. While unit testing allows you to test Kotlin/JVM
compilation, I have not found anything for unit testing Kotlin/Native compilation.

---

Thanks for reading my series of articles about Kotlin IR and writing a Kotlin compiler plugin! If
you would like to explore a simple, working Kotlin compiler plugin created for this article series,
you can check out the [debuglog] repository.

[Kotlin compatibility]: https://kotlinlang.org/docs/reference/evolution/components-stability.html
[Part 1]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1
[Part 2]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-2
[Part 3]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-3
[Part 4]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-4
[Part 5]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-5
[Part 6]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-6
[part-1-gradle-plugin]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-1#gradle-plugin
[part-5-transformer]: https://blog.bnorm.dev/writing-your-second-compiler-plugin-part-5#autobots-roll-out
[KT-43385]: https://youtrack.jetbrains.com/issue/KT-43385
[plugin-publish]: https://plugins.gradle.org/docs/publish-plugin
[plugin-publish-steps]: https://plugins.gradle.org/docs/submit
[bintray]: https://bintray.com/
[nexus]: https://oss.sonatype.org/
[nexus-setup]: https://central.sonatype.org/pages/ossrh-guide.html
[gradle-maven-publish-plugin]: https://github.com/vanniktech/gradle-maven-publish-plugin
[exhaustive]: https://github.com/cashapp/exhaustive/blob/trunk/gradle/publish.gradle
[redacted-compiler-plugin]: https://github.com/ZacSweers/redacted-compiler-plugin/blob/main/build.gradle
[kotlin-power-assert]: https://github.com/bnorm/kotlin-power-assert/blob/master/kotlin-power-assert-plugin/build.gradle.kts#L55
[composite builds]: https://docs.gradle.org/current/userguide/composite_builds.html
[GitHub template]: https://github.com/bnorm/kotlin-ir-plugin-template
[debuglog]: https://github.com/bnorm/debuglog
[debuglog-integration]: https://github.com/bnorm/debuglog/tree/main/debuglog-integration
