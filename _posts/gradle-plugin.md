title: Gradle plugin
tags:
  - gradle
  - 摘录
date: 2015-02-02 13:23:15
---

Gradle plugin packages up reusable pieces of build logic, which can be used across many different projects and builds.

Packaging a plugin
===================

Build script
------------

You can include the source for the plugin directly in the build script. This has the benefit that the plugin is automatically compiled and included in the classpath of the build script without you having to do anything. However, the plugin is not visible outside the build script, and so you cannot reuse the plugin outside the build script it is defined in.

<!--more-->

buildSrc project
----------------
You can put the source for the plugin in the rootProjectDir/buildSrc/src/main/groovy directory. Gradle will take care of compiling and testing the plugin and making it available on the classpath of the build script. The plugin is visible to every build script used by the build. However, it is not visible outside the build, and so you cannot reuse the plugin outside the build it is defined in.

See Chapter 60, Organizing Build Logic for more details about the buildSrc project.

Standalone project
-------------------
You can create a separate project for your plugin. This project produces and publishes a JAR which you can then use in multiple builds and share with others. Generally, this JAR might include some custom plugins, or bundle several related task classes into a single library. Or some combination of the two.


Writing a simple plugin
========================
build.gradle
```
apply plugin: GreetingPlugin

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') << {
            println "Hello from the GreetingPlugin"
        }
    }
}
```
Output of ***gradle -q hello***
`
> gradle -q hello
Hello from the GreetingPlugin
`

A standalone project
--------------------
Now we will move our plugin to a standalone project, so we can publish it and share it with others. This project is simply a Groovy project that produces a JAR containing the plugin classes.

So how does Gradle find the Plugin implementation? The answer is you need to provide a properties file in the jar's META-INF/gradle-plugins directory that matches the id of your plugin.

***Example 59.6. Wiring for a custom plugin***

src/main/resources/META-INF/gradle-plugins/org.samples.greeting.properties
```implementation-class=org.gradle.GreetingPlugin```


Using your plugin in another project
=====================================
To use a plugin in a build script, you need to add the plugin classes to the build script's classpath. To do this, you use a “buildscript { }” block, as described in [Section 60.5, “External dependencies for the build script”][2].

***Example 59.7. Using a custom plugin in another project***

build.gradle
```
buildscript {
    repositories {
        maven {
            url uri('../repo')
        }
    }
    dependencies {
        classpath group: 'org.gradle', name: 'customPlugin',
                  version: '1.0-SNAPSHOT'
    }
}
apply plugin: 'org.samples.greeting'
```

[1]: http://www.gradle.org/docs/current/userguide/custom_plugins.html
[2]: http://www.gradle.org/docs/current/userguide/organizing_build_logic.html#sec:external_dependencies