This specification outlines the work that is required to use Gradle to build applications that use the [Play framework](http://www.playframework.com).

# Use cases

There are 3 main use cases:

- A developer builds a Play application.
- A developer runs a Play application during development.
- A deployer runs a Play application. That is, a Play application is packaged up as a distribution which can be run in a production environment.

# Out of scope

The following features are currently out of scope for this spec, but certainly make sense for later work:

- Building a Play application for multiple Scala versions. For now, the build for a given Play application will target a single Scala version.
  It will be possible to declare which version of Scala to build for.
- Using continuous mode with JVMs older than Java 7. For now, this will work only with Java 7 and later will be supported. It will be possible to build and run for Java 6.
- Any specific IDE integration, beyond Gradle's current general purpose IDE integration for Java and Scala.
- Any specific testing support, beyond Gradle's current support for testing Java and Scala projects.
- Any specific support for publishing and resolving Play applications, beyond Gradle's current general purpose capabilities.
- Any specific support for authoring plugins, beyond Gradle's current support.
- Installing the Play tools on the build machine.
- Migrating or importing SBT settings for a Play project.

# Performance

Performance should be comparable to SBT:

- Building and starting an application.
- Reload after a change.
- Executing tests for an application.

# Milestone 1

For the first milestone, a developer will be able to define, build and run a simple play application that consists
of routes and templates, Java and Scala sources, as well as Javascript, CSS, CoffeeScript and LESSCSS sources.

In this milestone we are not dealing with:
- Automatically rebuilding the play application when sources change
- 3rd party dependencies for Java, Scala or Javascript. (This may change).

The goal is to get something working pretty quickly, and work to improve the configurability and discoverability over time.

## Feature: Developer builds, tests and runs a basic Play application

### Story: Build author declares and builds a Play application component (✔︎)

Add a `play-application` plugin that provides Play application component:

```gradle
plugins {
    id 'play-application'
}

model {
    components {
        myapp(PlayApplicationSpec) 
    }
}
```

- ~~Running `gradle assemble` builds an empty Jar file.~~
- ~~Running `gradle components` shows some basic details about the Play application.~~
- In this story, no source files are supported.

#### Test cases

- ~~component report shows PlayApplicationSpec with~~ 
- ~~version info about~~
      - ~~play  (declared in the plugin)~~
      - ~~java  (picked current version for now)~~
- ~~assemble creates an empty jar file~~

### Story: Developer builds a basic Play application (✔︎)

When running `gradle assemble`, a Jar file will be built for the default template application generated by Play.

- Using hard-coded:
    - Locations of java/scala files
    - Locations of routes file
    - Locations of templates files
    - Scala version
    - Dependencies of Play ("com.typesafe.play:play_2.11:2.3.7")
    - Dependency of template compiler ("com.typesafe.play:twirl-compiler_2.11:1.0.2)
    - Dependency of routes compiler
- resolve different play dependencies from user-configured repositories

#### Implementation

- setup TwirlCompiler task type
- setup RoutesCompiler task type
- Compile routes to scala and java
- Compile templates to scala
- Compile all scala (app/*/*.{scala,java}, output of: conf/routes, output of: app/views/*.scala.html) files
- Output class files are part of assemble jar file

#### Test cases

- verify that generated scala template files exists
- verify that generated scala/java route files exists
- `gradle assemble` should trigger compile task and output jar should contain class files

### Story: Developer runs a basic Play application (✔︎)

Extend the Play support to allow the Play application to be executed.

- Running `gradle assemble` produces an executable Jar that, when executed, runs the Play application.
- Running `gradle run<ComponentName>` builds and executes the Play application.

At this stage, only the default generated Play application is supported, with a hard-coded version of Scala and Play.
          
#### Test cases

- verify that play app can be built and executed with play version 2.3.7 and 2.2.3
- Can configure port for launched PlayApp: default is 9000

#### Open issue

- Stopping the app with Ctrl-D?
- Testing stopping via keyboard interaction with Daemon (currently test is disabled)

### Story: Developer tests a basic Play application (✔︎)

- Add a `Test` test task per binary that runs the play application tests based on JUnit TestRunner against the binary
- The test sources are compiled against the binary and the `play-test` dependency based on play and scala version of the platform
- Can execute Play unit and integration tests
- Fails with a nice error message
- Wired into the `check` and `test` lifecycle tasks

#### Test cases
- Verify that running `gradle testBinary` and `gradle test` executes tests in the `/test` directory
- Verify output and reports generated for successful tests
- Verify output and reports generated for failing tests

#### Open issues

- Model test suites
- Pull `check` lifecycle task up into `LifecycleBasePlugin` and wire in all test tasks.

### Story: Developer builds, runs and tests a basic application for a specified version of Play (✔︎)

- Can build play application with Play 2.2.3 on Scala 2.10, and Play 2.3.7 on Scala 2.11

#### Test cases

- Verify building and running the 'activator new' app with Play 2.2.3 and Play 2.3.7

#### Open issues

- Compile Scala wrappers around different Play versions, and load these via reflection

### Story: Developer builds Play application distribution (✔︎)

Introduce some lifecycle tasks to allow the developer to package up the Play application. For example, the
developer may run `gradle stage` to stage the local application, or `gradle dist` to create a standalone distribution.

- Build distribution image and zips, as per `play stage` and `play dist`
- Default distributions are added for each play binary
- Developer can add additional content to a default distribution:
```
model {
    distributions {
        playBinary {
            contents {
                from "docs"
            }
        }
    }
}
```


#### Test cases
- Default distributions are added for each play binary
- Stage and Zip tasks are added for each distribution
- Lifecycle tasks for "stage" and "dist" are added to aggregate all distribution tasks
- Distribution base name defaults to distribution name, which defaults to play binary name
- Distribution archive name defaults to "[distribution-base-name]-[project.version].zip"
- Public assets are packaged in a separate jar with a classifier of "assets".
- Default distribution zip contains:
    - all content under a directory named "[distribution-base-name]-[project.version]"
    - jar and assets jar in "lib".
    - all runtime libraries in "lib".
    - application script and batch file in "bin".
    - application.conf and any secondary routes files in "conf".
    - any README file provided in conventional location "${projectDir}/README"
- content added to the distribution is also included in the zip
- application script and batch file will successfully run play:
    - can access a public asset
    - can access a custom route
    
#### Open Issues
- A Play distribution zip, by default, contains a shared/docs directory with the scaladocs for the application.  We'll need 
a scaladoc task wired in to duplicate this functionality.

## Feature: Developer builds Play application with custom Java, Scala, routes and templates

### Story: Developer includes Scala sources in Jvm library (✔︎)

Add a new 'scala-lang' plugin to add Scala language support implemented in the same manner as the JavaLanguagePlugin.

```gradle
plugins {
    id 'jvm-component'
    id 'scala-lang'
}
model {
    components {
        myLib(JvmLibrarySpec)
        myOtherLib(JvmLibrarySpec) {
            sources {
                scala {
                    source.srcDir "src/myOtherLib/myScala"
                }
                otherScala(ScalaSourceSet) {
                    source.srcDir "src/otherScala"
                }
            }
        }
    }
}
```

##### Test cases

- Scala source set(s) and locations are visible in the components report
- Uses scala sources from conventional location
- Can configure location of scala sources
- Build is incremental:
    - Changed in Scala comment does not result in rebuilding JVM library
    - Change in Scala compile properties triggers recompilation
    - Change in Scala platform triggers recompilation
    - Change in Scala ToolChain triggers recompilation
    - Removal of Scala source files removes classes for that source file, which are removed from JVM library
- Compile is incremental: consider sources A, B, C where B depends on C
    - Change A, only A is recompiled
    - Change B, only B is recompiled
    - Change C, only B & C are recompiled
    - Remove A, outputs for A are removed
- Can include both Java and Scala sources in lib: no cross-compilation

### Story: Developer configures Scala sources and Jvm resources for Play application

Extend the Play support to full model Scala sources and Jvm resources.

- A default ScalaLanguageSourceSet 'scala' will include java and scala source files in `app/controllers` and `app/models`.
- A default JvmResourceSet 'resources' will include files in `public` and `conf`
- Files can be included/excluded in the default source sets
- Additional source sets can be configured
- All source sets are listed in the component report for a play application
- All Scala source sets in a Play application are joint-compiled

```gradle
plugins {
    id 'play-application'
}
model {
    components {
        play(PlayApplicationSpec) {
            sources {
                scala {
                    source.include "extraStuff/**"
                }
                resources {
                    source.srcDir "src/assets"
                }
                extraScala(ScalaLanguageSourceSet) {
                    source.srcDir "src/extraScala"
                }
            }
        }
    }
}
```

#### Test cases

- Scala and Jvm source sets (and locations) are visible in the components report
- Can configure additional includes and/or source directories for default `scala` and `resources` source sets
- Can provide an additional Scala source set for a Play application, with dependencies on main sources
- Can provide an additional resources set for a Play application
- Build is incremental:
    - Changed in Scala comment does not result in rebuilding Play app
    - Change in Scala or Java source file triggers recompilation
    - Change in Scala or Java compile properties triggers recompilation
    - Change in Platform triggers recompilation
    - Change in ToolChain triggers recompilation
    - Removal of Scala/Java source files removes classes from the Play app

#### Open issues

- Use `LanguageTransform` from `scala-lang` plugin to configure scala compile task for source sets
- Use source set dependencies to determine of source sets should be joint compiled

### Story: Developer configures template sources for Play application

Add a TwirlSourceSet and permit multiple instances in a Play application

- Twirl sources show up in component report
- Allow Twirl source location to be configured
- Allow additional Twirl source sets to be configured
- Define a generated ScalaSourceSet as the output for each TwirlSourceSet compilation
    - Generated ScalaSourceSet will be one of the Scala compile inputs
    - Generated ScalaSourceSet should not be visible in components report
- Source generation and compilation should be incremental and remove stale outputs.

#### Open issues

- handle non html templates
- Ability for Twirl compiler to prefer Java types in generated sources (i.e. SBT enablePlugins(PlayJava))
- Verify that default imports are configured: https://github.com/playframework/playframework/blob/master/framework/src/build-link/src/main/java/play/TemplateImports.java
- Allow Twirl to be used in a non-play project

### Story: Developer defines routes for Play application

Add a RoutesSourceSet and permit multiple instances in a Play application

- Routes sources show up in component report
- Allow Routes source location to be configured
- Allow additional Routes source sets to be configured
- Define a generated ScalaSourceSet as the output for each RoutesSourceSet compilation
    - Generated ScalaSourceSet will be one of the Scala compile inputs
    - Generated ScalaSourceSet should not be visible in components report
- Source generation and compilation should be incremental and remove stale outputs.

#### Open issues

- handle .routes files
- Ability for Routes compiler to prefer Java types in generated sources (i.e. SBT enablePlugins(PlayJava))

## Feature: Basic support for assets in Play application

### Story: Developer includes static assets in Play application (✔︎)

Any files under `/public` in the application sources are available under `/assets` in the running application.

### Story: Developer includes compiled coffeescript assets in Play application (✔︎)

Add a coffee script plugin as well as JavaScriptSourceSet and CoffeeScriptSourceSets and permit multiple instances.

```gradle
plugins {
    id 'play-application'
    id 'play-coffeescript'
}

model {
    components {
        play(PlayApplicationSpec) {
            sources {
                extraCoffeeScript(CoffeeScriptSourceSet) {
                    sources.srcDir "src/extraCoffeeScript"
                }

                extraJavaScript(JavaScriptSourceSet) {
                    sources.srcDir "src/extraJavaScript"
                }
            }
        }
    }
}
```

- Default coffeescript sourceset should be "app/assets/**/*.coffee"
- Compiled coffeescript files will be added to the jar under "public"
- Default javascript sourceset should be "app/assets/**/*.js"
- Processed javascript sourceset should be added to the jar under "public"

#### Test cases
- Coffeescript and javascript sources are visible in the components report
- Coffeescript sources successfully compiled to javascript
- Compiled coffeescript is added to jar under "public"
- Javascript sources are copied directly into jar under "public"
- Can provide additional coffeescript sources
- Can provide additional javascript sources
- Build is incremental:
    - Change in coffeescript source triggers recompile
    - No change in coffeescript source does not trigger a recompile
    - Removal of generated javascript triggers recompile
    - Removal of coffeescript source files removes generated javascript

### Story: Developer uses minified javascript assets in Play application (✔︎)

Extend the basic JavaScript plugin to include minified javascript assets in the Play application.
Use the Google Closure Compiler to produce a minified version of all Javascript assets. The minified files
should be named `<original-name>.min.<original-extension>` (usually the extension will be `.js`).

```gradle
plugins {
    id 'play'
}

model {
    components {
        play(PlayApplicationSpec) {
            sources {
                extraJavaScript(JavaScriptSourceSet) {
                    sources.srcDir "src/extraJavaScript"
                }
            }
        }
    }
}
```

#### Test cases
- Any javascript file in `app/assets` is available in minified form in the app.
- Any javascript file in a configured JavaScriptSourceSet is available in minified form in the app.
- Any compiled coffeeScript source file is available in both non-minified and minified javascript forms. 
- Build is incremental:
    - Minifier is not executed when no source inputs have changed
    - Changed javascript source produces changed minified javasript
    - Changed coffeescript source produces changed minified javascript
    - Removal of javascript source removes minified javascript
    - Removal of coffeescript source removes both minified and non-minified javascript


## Feature: Developer chooses target Play, Scala and/or Java platform

### Story: Compilation and building is incremental with respect to changes in Play platform

### Story: Build author declares target Play and Scala platform for Play application

```gradle
model {
    components {
        play(PlayApplicationSpec) {
            platform play: "2.3.7", scala: "2.11"
        }
    }
}
```

- If not specified, Play major version number implies Scala version
    - Play 2.2.x -> Scala 2.10
    - Play 2.3.x -> Scala 2.11
- Java version is taken from version executing Gradle

#### Test cases

- For each supported Play version: 2.2.3, 2.3.7
    - Can assemble Play application
    - Can run Play application
    - Can test Play application
- Can build & test multiple Play application variants in single build invocation
    - `gradle assemble` builds all variants
    - `gradle test` tests all variants
- Play version should be visible in components report and dependencies reports.
- Most Play integration tests should be able to run against different supported platforms
    - Developer build should run against single version by default
    - CI should run against all supported versions (using '-PtestAllPlatforms=true')

### Story: Build author declares target Java platform for Play application

```gradle
model {
    components {
        play(PlayApplicationSpec) {
            platform play: "2.3.7", scala: "2.11", java: "1.8"
        }
    }
}
```

### Story: Developer configures target Java and Scala platforms for Jvm library

```gradle
plugins {
    id 'jvm-component'
    id 'scala-lang'
}
model {
    components {
        scalaLib(JvmLibrarySpec) {
            platform java: "1.6", scala: "2.10"
        }
    }
}
```

- Pull `platform(Object)` method up from `PlayApplicationSpec` to `PlatformAwareComponentSpec`
- Allow arbitrary `Map<String, String>` inputs to `platform()`, converting into a `PlatformRequirement`
- Add a `CompositePlatform` type that is returned by `PlatformResolver.resolve()`
    - Allow various typed `PlatformResolver` instances to plugin to the composite `PlatformResolver`
    - When resolving a play platform, return a composite platform with `PlayPlatform`, `ScalaPlatform` and `JavaPlatform` components.
    - When resolving a jvm platform, return a composite platform with `JavaPlatform` and optional `ScalaPlatform` components.
         - Maybe add a `JvmPlatform` that extends `CompositePlatform`, encoding jvm compatibility version.
         - Maybe make `PlayPlatform` extend `CompositePlatform` with a `JvmPlatform` component. Later will have a `BrowserPlatform` component.
- Scala-lang plugin registers a resolver that returns a `ScalaPlatform` for a 'scala' platform requirement

### Story: New target platform DSL for native components

```gradle
model {
    components {
        nativeLib(NativeLibrarySpec) {
            platform os: "windows", arch: "x86"
        }
    }
}
```

## Feature: Basic dependency management for Play

### Story: Developer configures dependencies for Play Application

# Milestone 2

## Feature: Gradle continuous mode

This story adds a general-purpose mechanism which is able to keep the output of some tasks up-to-date when source files change. 
For example, a developer may run `gradle --watch <tasks>`.

When run in continuous mode, Gradle will execute a build and determine any files that are inputs to that build.
Gradle will then watch for changes to those input files, and re-execute the build when any file changes.

Input files are determined as:
- Files that are inputs to a task but not outputs of some other task
- Files that are inputs to the model

So:

- `gradle --watch run` would build and run the Play application. When a change to the source files are detected, Gradle would rebuild and
  restart the application.
- `gradle --watch test run` would build and run the tests and then the Play application. When a change to the source files is detected,
  Gradle would rerun the tests, rebuild and restart the Play application.
- `gradle --watch test` would build and run the tests. When a source file changes, Gradle would rerun the tests.

Note that for this feature, the implementation will assume that any source file affects the output of every task listed on the command-line.
For example, running `gradle --watch test run` would restart the application if a test source file changes.

### Story: Add continuous Gradle mode triggered by timer

Gradle will be able to start, run a set of tasks and then wait for a retrigger before re-executing the build.

#### Implementation

See spike: https://github.com/lhotari/gradle/commit/969510762afd39c5890398e881a4f386ecc62d75

- Gradle CLI/client process connects to a daemon process as normal.
- Gradle Daemon knows when we're in "continuous mode" and repeats the last build until cancelled.
- Gradle CLI waits for the build to finish (as normal).  
- Instead of returning after each build, the daemon goes into a retry loop until cancelled triggered by something.
- Possibility that this will work in non-daemon mode too (but not a goal)
- Initial implementation will use a periodic timer to trigger the build.
- Add new command-line option (`--watch`) 
    - Initially stuff this in `StartParameter`, but a new "Parameters" class might make sense
- `InProcessBuildExecutor` changes to understand "continuous mode" or wraps an existing `BuildController`
    - Similar to what the spike did
    - Build loop creates a new `GradleLauncher` or resets an existing `GradleLauncher` (if that's safe)
    - After build, executor waits for a trigger from somewhere else
- Create `TriggerService` 
    - Register `TriggerAction`s for a given trigger type
    - Trigger an action (async)
    - Send a `Trigger` to listeners

```
pseudo:

interface Trigger {}
interface TriggerAction extends Action<Trigger> {}
interface TriggerService {
    void addTriggerAction(TriggerAction action)
    void trigger(Trigger t)
}

triggerService.addTriggerAction({ trigger ->
    logger.log("Rebuilding due to: $trigger.reason")
    triggerBuild()
})

// In run/execute()
while (not cancelled) {
    launcher.run()
    waitForTrigger()
}
```

#### Test Coverage

- If Gradle build succeeds, we wait for trigger and print some sort of helpful message.
- If Gradle build fails, we still wait for trigger.
- If Gradle fails to start the first time, Gradle exits and does not wait for the trigger (e.g., invalid command-line or build script errors).
- On Ctrl+C, Gradle exits and daemon cancels build.
- When "trigger" is tripped, a build runs.
- Some kind of performance test that re-runs the build multiple times and looks for leaks (increasing number of threads)?

#### Open Issues

- Integrate with the build-announcements plugin, so that desktop notifications can be fired when something fails when rerunning the tasks.
- What does the output look like when we fail?
- OK to reuse "global client services" created in BuildActionsFactory across multiple builds

### Story: Continuous Gradle mode triggered by file change

Gradle will be able to start, run a set of tasks and then monitor one file for changes without exiting.  When this file is changed, the same set of tasks will be re-run.

#### Implementation

- Fail when this is enabled on Java 6 builds, tell the user this is only supported for Java 7+.
- Watch project directory for changes to trigger re-run
- Add `InputWatchService` that can be given Files to watch
- TODO: Figure out where this watch service lives (in CLI process or Daemon)
- When files change, mark the file as out of date
- Re-run trigger polls the watch service for changes at some default rate.  We should allow this to be adjusted (e.g., --watch=1s).
- Ignore build/ .gradle/ etc files.

#### Test Coverage

- When the project directory files change/are create/are delete, Gradle re-runs the same set of tasks.
- Limits/performance tests for watched files?

#### Open Issues

- The implementation for Java 1.7 `java.nio.file.WatchService` in Java 7 or even Java 8 isn't using a native file notification OS API on MacOSX. ([details](http://stackoverflow.com/questions/9588737/is-java-7-watchservice-slow-for-anyone-else), [JDK-7133447]( https://bugs.openjdk.java.net/browse/JDK-7133447), [openjdk mailing list](http://mail.openjdk.java.net/pipermail/nio-dev/2014-August/002691.html)) This doesn't scale to 1000s of input files. There are several [native file notification OS API wrappers for Java](http://wiki.netbeans.org/NativeFileNotifications) if performance is an issue. However there isn't a well-maintained file notification library with a suitable license. Play framework uses [JNotify](http://jnotify.sourceforge.net) [for native file notifications on MacOSX](https://github.com/playframework/playframework/blob/ca664a7/framework/src/run-support/src/main/scala/play/runsupport/FileWatchService.scala#L77-L88). JNotify is a dead project hosted in Sourceforge and not even available in maven central or jcenter. It looks like it's only available in Typesafe's maven repository.
    
- Do we want to support Java 1.6 with a polling implemention or just show an error when running on pre Java 1.7 ? Play framework uses [polling implementation on pre Java 1.7](https://github.com/playframework/playframework/blob/ca664a7/framework/src/run-support/src/main/scala/play/runsupport/FileWatchService.scala#L109) .

### Story: Continuous Gradle mode triggered by task input changes

Gradle will be able to start, run a set of tasks and then monitor changes to any inputs for the given tasks without exiting.  When any input file is changed, all of the same set of tasks will be re-run.

#### Implementation

- Add inputs to task to `InputWatchService` so that they can be watched

#### Test Coverage

- TBD

#### Open Issues

- Monitor files that are inputs to the model for changes too.

### Story: Continuous Gradle mode rebuilds if an input file is modified by the user while a build is running

#### Implementation

- IDEA: Outputs from tasks should be excluded as inputs to be watched (since they are 'intermediate' files)
- IDEA: Start/stop watching during a build?

#### Test Coverage

- TBD

#### Open Issues

- TBD

### Additional Notes 

TODO: See if these make sense incorporated in another story/Feature.

- When the tasks start a deployment, stop the deployment before rebuilding, or reload if supported by the deployment.
- If previous build started any service, stop that service before rebuilding.
- Uses Gradle daemon to run build.
- Collect up all input files as build runs.
- Deprecate reload properties from Jetty tasks, as they don't work well and are replaced by this general mechanism.

## Feature: Keep running Play application up-to-date when source changes

### Story: Domain model for Web Applications

This story generalises the current 'PlayRun' task, adding a basic web-application + deployment domain model and some lifecycle tasks
associated with this domain model.

Note that this story does not address reloading the application when source files change. This is addressed by a later story.

#### Implementation

```gradle
model {
    deployments {
        // one per Play application
        play(WebApplicationDeployment) {
            httpPort
            forkOptions
        }
    }
}
```

Web application plugin:

- Defines the concept of a 'web application'.
- Defines the concept of a 'web deployment': a web application hosted by a web server
- Defines `run` lifecycle task for a deployment

Play plugin:

- Defines a Play application as-a web application
- Defines a ${deployment}Run task for starting and deploying application.
- Configures `run` to depend on ${deployment}Run and ${deployment}Run to depend on the output of the play application.

When the Run task is executed for a deployment, Gradle will:

- Build the web application
- Start the web server
- Deploy the web application to the web server
- Add the running deployment to the container of 'running deployments' for the build

#### Open issues

- Modelling for deployment hosts
- Modelling of more general 'long-lived processes'

### Story: Gradle build stops any running deployments on exit

At the end of the build, Gradle will check to see if there are any running deployments. 
If so, it will wait for Ctrl-C before stopping each deployment and exiting.
This will replace the current `PlayRun` implementation of Gradle with general-purpose infrastructure.

When run in continuous mode, all running deployments should be stopped before re-executing the build.

### Story: Gradle does not restart reloadable deployments when re-executing a build in continuous mode

Integrate deployments with continuous mode, so that if a build completes with a running deployment then that
deployment is not stopped and restarted when the build re-executes due to input file change.

This should only apply for deployments that indicate that they are 'reloadable'. For now, the Play application
deployment implementation will not be reloadable.

#### Open Issues

- New reloadable Jetty plugin that will reload changed content when run in continuous mode

### Story: Play application reloads content on browser refresh

Using a BuildLink implementation, allow the Play application deployment to be 'reloadable', and to automatically 
reload the content on browser refresh.

At this stage, the build will be re-executed whenever an input file changes, not only when requested by
the BuildLink API.

#### Implementation

IDEA: Use Tooling API to allow Play BuildLink to connect to a build in continuous mode and:

- Receive events when the build is re-executed, including build failures

The mechanism will depend on the Play [build-link](https://repo.typesafe.com/typesafe/releases/com/typesafe/play/build-link/) library,
to inform Gradle when the application needs to be reloaded. 
See [Play's BuildLink.java](https://github.com/playframework/playframework/blob/master/framework/src/build-link/src/main/java/play/core/BuildLink.java)
for good documentation about interfacing between Play and the build system. 
Gradle will implement the `BuildLink` interface and provide it to the application hosting NettyServer. 
When a new request comes in, the Play application will call `BuildLink.reload` and Gradle return a new ClassLoader containing the rebuilt application to Play.
If the application is up-to-date, `BuildLink.reload` can return `false`.

See the SBT implementation of this logic, which may be a helpful guide:

Play 2.2.x implementation:
See
[PlayRun](https://github.com/playframework/playframework/blob/2.2.x/framework/src/sbt-plugin/src/main/scala/PlayRun.scala)
and [PlayReloader](https://github.com/playframework/playframework/blob/2.2.x/framework/src/sbt-plugin/src/main/scala/PlayReloader.scala)

Play master branch implementation:
See [PlayRun](https://github.com/playframework/playframework/blob/master/framework/src/sbt-plugin/src/main/scala/play/sbt/run/PlayRun.scala),
[PlayReload](https://github.com/playframework/playframework/blob/master/framework/src/sbt-plugin/src/main/scala/play/sbt/run/PlayReload.scala) and
[Reloader](https://github.com/playframework/playframework/blob/master/framework/src/run-support/src/main/scala/play/runsupport/Reloader.scala)

### Story: Play application triggers rebuild on browser refresh

Instead of rebuilding the Play application on every source file change, the application should be rebuilt only if
and input file has changed AND the BuildLink API requests a reload.

??? Not sure if this will be required.

## Feature: Developer views compile and other build failures in Play application

### Story: Developer views build failure message in Play application

Adapt a generic build failure exception to a `PlayException` that renders the exception message.

### Story: Developer views Java and Scala compilation failure in Play application

Adapt compilation failures so that the failure and content of the failing file is displayed in the Play application.

### Story: Developer views Asset compilation failures in Play application

Failures in CoffeeScript compilation are rendered with content of the failing file. 
This mechanism will be generally applicable to custom asset compilation tasks.

### Story: Developer views build failure stack trace in Play application

## Feature: Resources are built on demand when running Play application

When running a Play application, start the application without building any resources. Build these resources only when requested
by the client.

- On each request, check whether the task which produces the requested resource has been executed or not. If not, run the task synchronously
  and block until completed.
- Include the transitive input of these tasks as inputs to the watch mechanism, so that further changes in these source files will
  trigger a restart of the application at the appropriate time.
- Failures need to be forwarded to the application for display.

## Feature: Long running compiler daemon

Reuse the compiler daemon across builds to keep the Scala compiler warmed up. This is also useful for the other compilers.

### Implementation

- Maintain a registry of compiler daemons in ~/.gradle
- Daemons expire some time after build, with much shorter expiry than the build daemon.
- Reuse infrastructure from build daemon.


# Later milestones

## Feature: Developer adds default Play repositories to build script

This allows the developer to easily add repositories to the build script using a convenience extension on the RepositoryContainer object.

```gradle
buildscript {
    repositories {
        gradlePlay()
    }
}
```
 
This should point to a virtual repository (play-public) at gradle.repo.org that's backed by the default repositories required for play functionality.
Currently the following repositories would be required:
- https://repo.typesafe.com/typesafe/maven-releases (play support)
- https://repo.gradle.org/gradle/javascript-public (coffeescript and other javascript artifacts)

#### Test Cases
- Can build a basic play application using the convenience extension to specify repositories
- Can build a coffeescript-enabled play application using the convenience extension to specify repositories

## Feature: Build author provides plugin for asset processing in Play application (Javascript, LESS, CoffeeScript)

Extend the standard build lifecycle to compile the front end assets to CSS and Javascript.

- Provide some built-in implementations of asset plugins
    - Coffeescript -> Javascript
    - LESSCSS -> CSS
    - Javascript > Javascript via Google Closure
    - Javascript minification, requirejs optimization
- For built-in asset plugins
    - Build on public API
    - Include the compiled assets in the Jar
    - Define source sets for each type of source file
    - Compilation should be incremental and remove stale outputs
    - Expose some compiler options

### Implementation

JavaScript language plugin:

- Defines JavaScript library component and associated JavaScript bundle binary.
- Defines JavaScript source set type (a JavaScript bundle and JavaScript source set should be usable in either role).
- Defines transformation from JavaScript source set to JavaScript bundle.

CSS language plugin:

- Defines CSS library component and associated CSS bundle binary.
- Defines CSS source set type (a CSS bundle and CSS source set should be usable in either role).
- Defines transformation from CSS source set to CSS bundle.

CoffeeScript plugin:

- Defines CoffeeScript source set type and transformation to JavaScript bundle.

LESSCSS plugin:

- Defines LESSCSS source set type and transformation to CSS bundle.

Google Closure plugin:

- Defines transformation from JavaScript source set to JavaScript bundle.

Play plugin:

- Defines JavaScript and CSS components for the Play application.
- Wires in the appropriate outputs to assemble the Jar.

### Open issues

- Integration with existing Gradle javascript plugins.

## Documentation

- Migrating an SBT based Play project to Gradle
- Writing Gradle plugins that extend the base Play plugin

## Native integration with Specs 2

Introduce a test integration which allows Specs 2 specifications to be executed directly by Gradle, without requiring the use of the Specs 2
JUnit integration.

- Add a Specs 2 plugin
- Add some Specs 2 options to test tasks
- Detect specs 2 specifications and schedule for execution
- Execute specs 2 specifications using its API and adapt execution events

Note: no changes to the HTML or XML test reports will be made.

## Developer runs Scala interactive console

Allow the Scala interactive console to be launched from the command-line.

- Build the project's main classes and make them visible via the console
- Add support for client-side execution of actions
- Model the Scala console as a client-side action
- Remove console decoration prior to starting the Scala console

## Scala code quality plugins

- Scalastyle
- SCCT

## Javascript plugins

- Compile Dust templates to javascript and include in the web application image

## Bootstrap a new Play project

Extend the build init plugin so that it can bootstrap a new Play project, producing the same output as `activator new` except with a Gradle build instead of
an SBT build.

## Publish Play application to a binary repository

Allow a Play application distribution to be published to a binary repository.

Some candidates for later work:

- Improve the HTML test report to render a tree of test executions, for better reporting of Specs 2 execution (and other test frameworks)
- Support the new Java and Scala language plugins
- Improve watch mode so that only those tasks affected be a given change are executed

## Further features

- Internal mechanism for plugin to inject renderer(s) into components report.
- Model language transformations, and change Play support to allow a Play application to take any JVM language as input.
- Declare dependencies on other Java/Scala libraries
- Control joint compilation of sources based on source set dependencies.
- Build multiple variants of a Play application.
- Generate an application install, eg with launcher scripts and so on.
