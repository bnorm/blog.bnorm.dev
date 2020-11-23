---
layout: post
title: Writing Your Second Kotlin Compiler Plugin, Part 1 — Project Setup
tweet: https://twitter.com/bnormcodes/status/1330241400391262209?s=20
---

> At the time of writing this article, [Kotlin compatibility] for IR backend is in Alpha status and
> the compiler plugin API is Experimental. As such, information contained in this article about IR
> and compiler plugins could be out-of-date or incorrect. If official documentation exists, please
> refer to it first.

I've been wanting to write this series of blog posts for a while. Inspired By Kevin Most's 2018
KotlinConf talk - [Writing Your First Kotlin Compiler Plugin] - I wanted to show how we could write
our second compiler plugin, something IR based that works on all Kotlin targets.

In this first part we will go over the project setup required to build a compiler plugin. In later
parts we will explore navigating and transforming Kotlin IR.

- Part 2 - Navigating Kotlin IR
- Part 3 - Transforming Kotlin IR
- Part 4 - ?
- Part ? - Publishing a Kotlin Compiler Plugin

# Setup

There are a number pieces to a Kotlin compiler plugin. At least 3 modules are required if you want
to support all Kotlin targets:

- Gradle plugin which adds your plugin to Kotlin compiler classpath,
- Kotlin compiler plugin for non-native Kotlin platforms,
- Kotlin compiler plugin for Kotlin/Native platform.

Kotlin/Native requires a separate compiler plugin because certain classes used by the Kotlin
compiler have a different package on Kotlin/Native. We'll explore this oddity more in-depth later.

# Gradle Plugin

The [Gradle plugin] part of a Kotlin compiler plugin project is responsible for defining a few things:

- Artifact coordinates of the Kotlin compiler plugin,
- ID string of the Kotlin compiler plugin,
- Translation of Gradle configuration to command line options.

The artifact coordinates are used to download the correct compiler plugin artifact from MavenCentral
(or other location) to include on the compiler plugin classpath. The compiler plugin ID string is
used to separate command line options by plugin to avoid conflicting options between multiple
plugins.

The artifact coordinates and ID we'll [define with Gradle] using the [buildconfig] plugin. Since the
artifact coordinates - group, name, and version - are already defined by Gradle and the Kotlin
compiler plugin ID will be the Gradle plugin ID, this will make things consistent when everything is
published.

```kotlin
buildConfig {
  val project = project(":kotlin-ir-plugin")
  packageName(project.group.toString())
  buildConfigField("String", "KOTLIN_PLUGIN_ID", "\"${rootProject.extra["kotlin_plugin_id"]}\"")
  buildConfigField("String", "KOTLIN_PLUGIN_GROUP", "\"${project.group}\"")
  buildConfigField("String", "KOTLIN_PLUGIN_NAME", "\"${project.name}\"")
  buildConfigField("String", "KOTLIN_PLUGIN_VERSION", "\"${project.version}\"")
}
```

These buildconfig values are used by a [`KotlinCompilerPluginSupportPlugin`] implementation which is
responsible for bridging between Gradle and the Kotlin compiler.

```kotlin
class TemplateGradlePlugin : KotlinCompilerPluginSupportPlugin {
  override fun apply(target: Project): Unit = with(target) {
    extensions.create("template", TemplateGradleExtension::class.java)
  }

  override fun isApplicable(kotlinCompilation: KotlinCompilation<*>): Boolean = true

  override fun getCompilerPluginId(): String = BuildConfig.KOTLIN_PLUGIN_ID

  override fun getPluginArtifact(): SubpluginArtifact = SubpluginArtifact(
    groupId = BuildConfig.KOTLIN_PLUGIN_GROUP,
    artifactId = BuildConfig.KOTLIN_PLUGIN_NAME,
    version = BuildConfig.KOTLIN_PLUGIN_VERSION
  )

  override fun getPluginArtifactForNative(): SubpluginArtifact = SubpluginArtifact(
    groupId = BuildConfig.KOTLIN_PLUGIN_GROUP,
    artifactId = BuildConfig.KOTLIN_PLUGIN_NAME + "-native",
    version = BuildConfig.KOTLIN_PLUGIN_VERSION
  )

  override fun applyToCompilation(
    kotlinCompilation: KotlinCompilation<*>
  ): Provider<List<SubpluginOption>> {
    val project = kotlinCompilation.target.project
    val extension = project.extensions.getByType(TemplateGradleExtension::class.java)
    return project.provider {
      listOf(
        SubpluginOption(key = "string", value = extension.stringProperty.get()),
        SubpluginOption(key = "file", value = extension.fileProperty.get().asFile.path),
      )
    }
  }
}
```

Any [custom Gradle properties] defined in a [Gradle extension] are translated to Kotlin compiler
arguments using the `applyToCompilation` function. These will be read by our Kotlin compiler plugin
which is next!

# Kotlin Plugin

A Kotlin compiler plugin starts with a `ComponentRegistrar` and `CommandLineProcessor`
implementation. Both of these classes are loaded by the Kotlin compiler using a [ServiceLoader] so
we will use the [`auto-service`] annotation processer to automatically generate the required files.

The [`CommandLineProcessor`] is pretty easy to understand: It defines the Kotlin compiler plugin ID
string (same as in Gradle plugin) and expected command line options. The processor is also
responsible for parsing the command line options of the plugin and placing them in a 
`CompilerConfiguration`. It should read and process the same values defined by the Gradle plugin.

```kotlin
@AutoService(CommandLineProcessor::class)
class TemplateCommandLineProcessor : CommandLineProcessor {
  companion object {
    private const val OPTION_STRING = "string"
    private const val OPTION_FILE = "file"

    val ARG_STRING = CompilerConfigurationKey<String>(OPTION_STRING)
    val ARG_FILE = CompilerConfigurationKey<String>(OPTION_FILE)
  }

  override val pluginId: String = BuildConfig.KOTLIN_PLUGIN_ID

  override val pluginOptions: Collection<CliOption> = listOf(
    CliOption(
      optionName = OPTION_STRING,
      valueDescription = "string",
      description = "sample string argument",
      required = false,
    ),
    CliOption(
      optionName = OPTION_FILE,
      valueDescription = "file",
      description = "sample file argument",
      required = false,
    ),
  )

  override fun processOption(
    option: AbstractCliOption,
    value: String,
    configuration: CompilerConfiguration
  ) {
    return when (option.optionName) {
      OPTION_STRING -> configuration.put(ARG_STRING, value)
      OPTION_FILE -> configuration.put(ARG_FILE, value)
      else -> throw IllegalArgumentException("Unexpected config option ${option.optionName}")
    }
  }
}
```

The [`ComponentRegistrar`] is where the actual work of a compiler plugin actually begins. The
registrar is responsible for registering extension components with the project being compiled. There
are a number of extension points currently available in the Kotlin compiler, but the one we will be
focusing on is `IrGenerationExtension`, which allows for navigating and transforming Kotlin IR.

```kotlin
@AutoService(ComponentRegistrar::class)
class TemplateComponentRegistrar(
  private val defaultString: String,
  private val defaultFile: String,
) : ComponentRegistrar {

  @Suppress("unused") // Used by service loader
  constructor() : this(
    defaultString = "Hello, World!",
    defaultFile = "file.txt"
  )

  override fun registerProjectComponents(
    project: MockProject,
    configuration: CompilerConfiguration
  ) {
    val messageCollector = configuration.get(CLIConfigurationKeys.MESSAGE_COLLECTOR_KEY, MessageCollector.NONE)
    val string = configuration.get(TemplateCommandLineProcessor.ARG_STRING, defaultString)
    val file = configuration.get(TemplateCommandLineProcessor.ARG_FILE, defaultFile)

    IrGenerationExtension.registerExtension(project, TemplateIrGenerationExtension(messageCollector, string, file))
  }
}
```

An instance of [`IrGenerationExtension`] can be created and registered within a
`ComponentRegistrar`. This instance will be called during compilation when IR code needs to be
generated (or transformed). The entry point for this extension provides an `IrModuleFragment` and
`IrPluginContext` which provide everything we need to navigate and transform the module IR code.

```kotlin
class TemplateIrGenerationExtension(
  private val messageCollector: MessageCollector,
  private val string: String,
  private val file: String
) : IrGenerationExtension {
  override fun generate(moduleFragment: IrModuleFragment, pluginContext: IrPluginContext) {
    messageCollector.report(CompilerMessageSeverity.INFO, "Argument 'string' = $string")
    messageCollector.report(CompilerMessageSeverity.INFO, "Argument 'file' = $file")
  }
}
```

We'll be going more into how to use the provided `IrModuleFragment` and `IrPluginContext` values in
the next parts of this series. But next we need to talk about how Kotlin/Native is different.

# Kotlin Plugin - Native

You may have noticed that the Gradle plugin defines a separate coordinate for the Kotlin/Native
compiler plugin artifact. This is because Kotlin/Native compiler plugins are compiled with a
different dependency than other compiler plugins. [Kotlin/Native compiler plugins] use the
`org.jetbrains.kotlin:kotlin-compiler` artifact while [plugins for every other platform] use the
`org.jetbrains.kotlin:kotlin-compiler-embedded` artifact.

The only difference I have found between these artifacts is the embedded artifact shadows some
classes so their package is different. Other then that, the code between these plugins can be
exactly the same. To share code, I use [Gradle to copy the code] of the non-Kotlin/Native compiler
plugin over to the Kotlin/Native compiler plugin module and change the import statements as
required.

```kotlin
tasks.named("compileKotlin") { dependsOn("syncSource") }
tasks.register<Sync>("syncSource") {
  from(project(":kotlin-ir-plugin").sourceSets.main.get().allSource)
  into("src/main/kotlin")
  filter {
    // Replace shadowed imports from plugin module
    when (it) {
      "import org.jetbrains.kotlin.com.intellij.mock.MockProject" -> "import com.intellij.mock.MockProject"
      else -> it
    }
  }
}
```

This isn't ideal, but I'm sure this oddity will be resolved as Kotlin/Native and compiler plugins
mature.

# Kotlin Plugin - Testing

Testing Kotlin/JVM compiler plugins is really easy thanks to the [`kotlin-compile-testing`] library!
This library allows you to [compile Kotlin source strings] with your ComponentRegistrar in tests
which makes debugging easy. The resulting compiled files can even be loaded via a ClassLoader if you
want to execute them which we will discuss later.

```kotlin
class IrPluginTest {
  @Test
  fun `IR plugin success`() {
    val result = compile(
      sourceFile = SourceFile.kotlin(
        "main.kt", """
fun main() {
  println(debug())
}
fun debug() = "Hello, World!"
"""
      )
    )
    assertEquals(KotlinCompilation.ExitCode.OK, result.exitCode)
  }
}

fun compile(
  sourceFiles: List<SourceFile>,
  plugin: ComponentRegistrar = TemplateComponentRegistrar(),
): KotlinCompilation.Result {
  return KotlinCompilation().apply {
    sources = sourceFiles
    useIR = true
    compilerPlugins = listOf(plugin)
    inheritClassPath = true
  }.compile()
}

fun compile(
  sourceFile: SourceFile,
  plugin: ComponentRegistrar = TemplateComponentRegistrar(),
): KotlinCompilation.Result {
  return compile(listOf(sourceFile), plugin)
}
```

If you execute the tests you should see the following output present in the log.

```
i: Argument 'string' = Hello, World!
i: Argument 'file' = file.txt
```

---

And with that, congrats! You now know how to set up a Kotlin compiler plugin project and are ready
to start navigating and transforming IR code! If you want to avoid the manual work of setting all of
this up, you can use the [GitHub template] I created when writing this article.


[Kotlin compatibility]: https://kotlinlang.org/docs/reference/evolution/components-stability.html
[Writing Your First Kotlin Compiler Plugin]: https://www.youtube.com/watch?v=w-GMlaziIyo
[Gradle plugin]: https://github.com/bnorm/kotlin-ir-plugin-template/tree/medium-part1/kotlin-ir-plugin-gradle
[define with Gradle]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin-gradle/build.gradle.kts
[buildconfig]: https://github.com/gmazzo/gradle-buildconfig-plugin
[`KotlinCompilerPluginSupportPlugin`]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin-gradle/src/main/kotlin/com/bnorm/template/TemplateGradlePlugin.kt
[custom Gradle properties]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin-gradle/src/main/kotlin/com/bnorm/template/TemplateGradleExtension.kt
[Gradle extension]: https://docs.gradle.org/current/userguide/custom_plugins.html
[ServiceLoader]: https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html
[`auto-service`]: https://github.com/google/auto/tree/master/service
[`CommandLineProcessor`]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin/src/main/kotlin/com/bnorm/template/TemplateCommandLineProcessor.kt
[`ComponentRegistrar`]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin/src/main/kotlin/com/bnorm/template/TemplateComponentRegistrar.kt
[`IrGenerationExtension`]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin/src/main/kotlin/com/bnorm/template/TemplateIrGenerationExtension.kt
[Kotlin/Native compiler plugins]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin-native/build.gradle.kts
[plugins for every other platform]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin/build.gradle.kts
[Gradle to copy the code]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin-native/build.gradle.kts#L16-L27
[`kotlin-compile-testing`]: https://github.com/tschuchortdev/kotlin-compile-testing
[compile Kotlin source strings]: https://github.com/bnorm/kotlin-ir-plugin-template/blob/medium-part1/kotlin-ir-plugin/src/test/kotlin/com/bnorm/template/IrPluginTest.kt
[GitHub template]: https://github.com/bnorm/kotlin-ir-plugin-template