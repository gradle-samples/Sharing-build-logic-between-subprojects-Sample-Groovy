# Sharing build logic between subprojects Sample Kotlin

NOTE: You can open this sample inside an IDE using the [IntelliJ native importer](https://www.jetbrains.com/help/idea/gradle.html#gradle_import_project_start) or [Eclipse Buildship](https://projects.eclipse.org/projects/tools.buildship).


This sample shows how build logic in a multi-project build can be organized into reusable plugins.

### Use case

As an example, let's say a project with three subprojects produces two public Java libraries that use the third subproject as an internal shared library.
This is the project structure:

```
├── internal-module
│   └── build.gradle.kts
├── library-a
│   ├── build.gradle.kts
│   └── README.md
├── library-b
│   ├── build.gradle.kts
│   └── README.md
└── settings.gradle.kts
```

Let's say that all our projects will be Java projects.
In this case we want to apply a common set of rules to all of them, such as source directory layout, compiler flags,
code style conventions, code quality checks and so on.

Two out of three projects are more than just Java projects - they are libraries that we perhaps want to publish to an
external repository. Publishing configuration, such as a common group name for the libraries as well as the repository coordinates
might be a cross-cutting concern that both libraries need to share. For this example let's also say that we want to enforce that
our libraries expose some documentation with a common structure.

### Organizing build logic

From the use case above, we have identified that we have two types of projects - generic Java projects and public libraries.
We can model this use case by layering two separate plugins that each define the type of project that applies them:

```
├── buildSrc
│   ├── build.gradle.kts
│   ├── settings.gradle.kts
│   ├── src
│   │   ├── main
│   │   │   └── kotlin
│   │   │       ├── myproject.java-conventions.gradle.kts
│   │   │       └── myproject.library-conventions.gradle.kts
...
```

* `myproject.java-conventions` - configures conventions that are generic for any Java project in the organization.
It applies the core `java` and `checkstyle` plugins as well as an external `com.github.spotbugs` plugin, configures common
compiler options as well as code quality checks.
* `myproject.library-conventions` - adds publishing configuration to publish to the organization's repository and checks for mandatory content in a README.
It applies `java-library` and `maven-publish` plugins as well as the `myproject.java-conventions` plugin.

The internal library subproject applies `myproject.java-conventions` plugin.

internal-module/build.gradle:

```groovy
plugins {
    id 'myproject.java-conventions'
}

dependencies {
    // internal module dependencies
}
```

The two public library subprojects apply `myproject.library-conventions` plugin.
library-a/build.gradle:
```groovy
plugins {
    id 'myproject.library-conventions'
}

dependencies {
    implementation project(':internal-module')
}
```

library-b/build.gradle:
```groovy
plugins {
    id 'myproject.library-conventions'
}

dependencies {
    implementation project(':internal-module')
}
```

Note how applying a convention plugin to a subproject effectively declares its type.
By applying `myproject.java-conventions` plugin we state: this is a "Java" project.
By applying `myproject.library-conventions` plugin we state: this is a "Library" project.

All plugins created in this sample contain functional tests that use [TestKit](https://docs.gradle.org/current/userguide/test_kit.html) to verify their behavior.

This sample does not have any project source code and only lays out a hypothetical project structure where two library subprojects depend on a shared internal subproject.


### Compiling convention plugins

In this sample, convention plugins are implemented as [precompiled script plugins](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:precompiled_plugins) -
this is the simplest way to start out as you can use one of Gradle's DSLs directly to implement the build logic, just as if the plugin was
a regular build script.

In order for precompiled script plugins to be discovered, the `buildSrc` project needs to apply the `groovy-dsl` plugin
in its `build.gradle` file.

buildSrc/build.gradle:
```groovy
plugins {
    id 'groovy-gradle-plugin'
}
```

### Things to note

#### Applying an external plugin in precompiled script plugin

The `myproject.java-conventions` plugin uses SpotBugs plugin to perform static code analysis.

SpotBugs is an external plugin - external plugins link:{userManualPath}/custom_plugins.html#applying_external_plugins_in_precompiled_script_plugins[need to be added as implementation dependencies] before they can be applied in a precompiled script plugin.

buildSrc/build.gradle:

```groovy
repositories {
    gradlePluginPortal() // so that external plugins can be resolved in dependencies section
}

dependencies {
    implementation 'gradle.plugin.com.github.spotbugs.snom:spotbugs-gradle-plugin:4.7.2'
    testImplementation platform("org.spockframework:spock-bom:2.1-groovy-3.0")
    testImplementation 'org.spockframework:spock-core'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

* The dependency artifact coordinates (GAV) for a plugin can be different from the plugin id.
* The Gradle Plugin Portal (`gradlePluginPortal()`) is added as a repository for plugin dependencies.
* The plugin version is determined from the dependency version.

Once the dependency is added, the external plugin can be applied in precompiled script plugin by id.

buildSrc/src/main/groovy/myproject.java-conventions.gradle:

```groovy
plugins {
    id 'java'
    id 'checkstyle'

    // NOTE: external plugin version is specified in implementation dependency artifact of the project's build file
    id 'com.github.spotbugs'
}
```

#### Applying other precompiled script plugins

Precompiled script plugins can apply other precompiled script plugins.

The `myproject.library-conventions` plugin applies the `myproject.java-conventions` plugin.

buildSrc/src/main/groovy/myproject.library-conventions.gradle:

```groovy
plugins {
    id 'java-library'
    id 'maven-publish'
    id 'myproject.java-conventions'
}
```

#### Using classes from the main source set

Precompiled script plugins can use classes defined in the main source set of the plugins project.

In this sample, `myproject.library-conventions` plugin uses a custom task class from `buildSrc/src/main/java` to configure library README checks.

buildSrc/src/main/groovy/myproject.library-conventions.gradle:

```groovy
def readmeCheck = tasks.register('readmeCheck', com.example.ReadmeVerificationTask) {
    // Expect the README in the project directory
    readme = layout.projectDirectory.file("README.md")
    // README must contain a Service API header
    readmePatterns = ['^## API$', '^## Changelog$']
}
```

For more details on authoring custom Gradle plugins, consult the link:{userManualPath}/custom_plugins.html[user manual].
