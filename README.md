# sbt-catalysts

[![Join the chat at https://gitter.im/typelevel/sbt-catalysts](https://badges.gitter.im/typelevel/sbt-catalysts.svg)](https://gitter.im/typelevel/sbt-catalysts?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Sbt plugin that provides a common build infrastructure and library dependencies for typelevel-like and 
similar cross platform projects. 

## Usage

To use it, just add it to your `project/plugins.sbt`:

```scala
addSbtPlugin("org.typelevel" % "sbt-catalysts" % "0.5.2")
```

This will automatically:

- Include other plugins - see [the build file for the current list](https://github.com/typelevel/sbt-catalysts/blob/master/build.sbt#L13-L25)
- Provide versions and library settings used by typelevel and related projects - see [Dependencies](https://github.com/typelevel/sbt-catalysts/blob/master/src/main/scala/org/typelevel/TypelevelDeps.scala)
- Include some functionality to help keep your build file sane - see the quick example below

The current list of dependencies and versions can be found here:

 - [Typelevel dependencies](https://github.com/typelevel/sbt-catalysts/blob/master/src/main/scala/org/typelevel/TypelevelDeps.scala#L15)

## G8 Template

The easiest way to create a new project using sbt-catalyst is through its g8 template.
```bash
sbt new typelevel/sbt-catalysts.g8
```
Or you can follow the example below, which is slightly different from the g8 template.

## Quick Example

In this example build file, we define a project that:
- Has a root project configured as an umbrella project for JVM/JS builds for multiple scala versions
- Has two sub-projects(modules) that cross compile to JVM and JS
- Use the standard typelevel set of dependencies and versions
- Provides best-practice scalac options
- Publishes regular snapshots as well as git-versioned immutable snapshots
- Sets the root project's console settings to include the required dependencies
- Includes Scoverage test coverage metrics
- Configured for automated release to Sonatype OSS release and snapshot repositories
- Generates scaladoc documentation and a GithubPages web site

```scala
import org.typelevel.Dependencies._

addCommandAlias("gitSnapshots", ";set version in ThisBuild := git.gitDescribedVersion.value.get + \"-SNAPSHOT\"")

/**
 * Project settings
 */
val gh = GitHubSettings(org = "InTheNow", proj = "catalysts", publishOrg = "org.typelevel", license = apache)
val devs = Seq(Dev("Alistair Johnson", "inthenow"))

val vAll = Versions(versions, libraries, scalacPlugins)

/**
 * Root - This is the root project that aggregates the catalystsJVM and catalystsJS sub projects
 */
lazy val rootSettings = buildSettings ++ commonSettings ++ publishSettings ++ scoverageSettings
lazy val module = mkModuleFactory(gh.proj, mkConfig(rootSettings, commonJvmSettings, commonJsSettings))
lazy val prj = mkPrjFactory(rootSettings)

lazy val rootPrj = project
  .configure(mkRootConfig(rootSettings,rootJVM))
  .aggregate(rootJVM, rootJS)
  .dependsOn(rootJVM, rootJS)

lazy val rootJVM = project
  .configure(mkRootJvmConfig(gh.proj, rootSettings, commonJvmSettings))
  .aggregate(macrosJVM, platformJVM, docs)
  .dependsOn(macrosJVM, platformJVM)

lazy val rootJS = project
  .configure(mkRootJsConfig(gh.proj, rootSettings, commonJsSettings))
  .aggregate(macrosJS, platformJS)

/** Macros - cross project that defines macros.*/
lazy val macros    = prj(macrosM)
lazy val macrosJVM = macrosM.jvm
lazy val macrosJS  = macrosM.js
lazy val macrosM   = module("macros", CrossType.Pure)
  .settings(macroCompatSettings(vAll):_*)

/** Platform - cross project that provides cross platform support.*/
lazy val platform    = prj(platformM)
lazy val platformJVM = platformM.jvm
lazy val platformJS  = platformM.js
lazy val platformM   = module("platform", CrossType.Dummy)
  .dependsOn(macrosM)
  .settings(addLibs(vAll, "specs2-core","specs2-scalacheck" ):_*)
  .settings(addTestLibs(vAll, "scalatest"):_*)

/** Docs - Generates and publishes the scaladoc API documents and the project web site.*/
lazy val docs = project.configure(mkDocConfig(gh, rootSettings, commonJvmSettings,
    platformJVM, macrosJVM))

/** Settings.*/
lazy val buildSettings = sharedBuildSettings(gh, vAll)

lazy val commonSettings = sharedCommonSettings ++ Seq(
  parallelExecution in Test := false
) ++ scalacAllSettings ++ unidocCommonSettings

lazy val commonJsSettings = Seq(scalaJSStage in Global := FastOptStage)

lazy val commonJvmSettings = Seq()
  
lazy val publishSettings = sharedPublishSettings(gh, devs) ++ credentialSettings ++ sharedReleaseProcess

lazy val scoverageSettings = sharedScoverageSettings(60)
```

## Overview

It is very important to realise that using the plugin does not actually change the
the build in any way, but merely provides the methods to help create a build.sbt. 
The design goal was thus to facilitate using SBT, rather than replace it.  

Compared to a plugin that "does everything", the advantage of this approach is that it is far
easier to use other functionality where appropriate and also to "see" what the build is
is actually doing, as the methods are (mainly) just pure SBT methods. 



Where we do deviate from "pure" SBT is how we define library dependencies. The norm would be:

In a small project, define the library dependencies explicitly:

```scala
libraryDependencies += "org.typelevel" %% "alleycats" %  "0.1.0"
```

If another sub-project also has this dependency, it's common to move the definition to a val, and
often the version too. It the library is used in another module for test only, another val is 
required:

```scala
val alleycatsV = "0.1.0"
val alleycatsDeps = Seq(libraryDependencies += "org.typelevel" %% "alleycats" % alleycatsV)
val alleycatsTestDeps = Seq(libraryDependencies += "org.typelevel" %% "alleycats" % alleycatsV % "test")
```

Whilst this works fine for individual projects, it's not ideal when a group of loosely coupled want
to share a common set (or sets) of dependencies, that they can also modify locally if required.
In this sense, we need "cascading configuration dependency files" and this is what this plugin also
provides.

"org.typelevel.depenendcies" provides two Maps, one for library versions the the for individual libraries
with their organisation, name and version. Being standard scala Maps, other dependency Maps can be added
with new or updated dependencies. The same applies for scala plugins. To use, we create the three Maps and
add to a combined container and then add the required dependencies to a module: eg

```scala
val vers = typelevel.versions ++ catalysts.versions + ("discipline" -> "0.3")
val libs = typelevel.libraries ++ catalysts.libraries
val addins = typelevel.scalacPlugins ++ catalysts.scalacPlugins
val vAll = Versions(vers, libs, addins)
....
.settings(addLibs(vAll, "specs2-core","specs2-scalacheck" ):_*)
.settings(addTestLibs(vAll, "scalatest" ):_*)
```

## Detailed Setup

To follow...

## Projects using sbt-catalysts

+ [alleycats][alleycats]
+ [mainecoon][mainecoon]
+ [catalysts][catalysts]

### Maintainers

The current maintainers (people who can merge pull requests) are:

 * [adelbertc](https://github.com/adelbertc) Adelbert Chang
 * [ochrons](https://github.com/ochrons) Otto Chrons
 * [BennyHill](https://github.com/BennyHill) Alistair Johnson
 * [non](https://github.com/non) Erik Osheim
 * [milessabin](https://github.com/milessabin) Miles Sabin
 * [fthomas](https://github.com/fthomas) Frank S. Thomas
 * [julien-truffaut](https://github.com/julien-truffaut) Julien Truffaut
 * [dwijnand](https://github.com/dwijnand) Dale Wijnand
 * [kailuowang](https://github.com/kailuowang) Kailuo Wang
 * [andyscott](https://github.com/andyscott) Andy Scott

 
We are currently following a practice of requiring at least two
sign-offs to merge PRs (and for large or contentious issues we may
wait for more). For typos or other small fixes to documentation we
relax this to a single sign-off.

### Contributing

Discussion around sbt-catalysts is currently happening in the
gitter channel, issue and PR pages.
You can get an overview of who is working on what
via [Waffle.io](https://waffle.io/typelevel/sbt-catalysts).

Feel free to open an issue if you notice a bug, have an idea for a
feature, or have a question about the code. Pull requests are also
gladly accepted.

People are expected to follow the
[Typelevel Code of Conduct](http://typelevel.org/conduct.html) when
discussing Catalysts on the Github page, Gitter channel, or other
venues.

We hope that our community will be respectful, helpful, and kind. If
you find yourself embroiled in a situation that becomes heated, or
that fails to live up to our expectations, you should disengage and
contact one of the [project maintainers](#maintainers) in private. We
hope to avoid letting minor aggressions and misunderstandings escalate
into larger problems.

If you are being harassed, please contact one of [us](#maintainers)
immediately so that we can support you.

### License

Catalysts is licensed under the **[Apache License, Version 2.0][apache]** (the
"License"); you may not use this software except in compliance with the License.

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

[apache]: https://www.apache.org/licenses/LICENSE-2.0
[alleycats]: https://github.com/non/alleycats
[catalysts]: https://github.com/typelevel/catalysts
[mainecoon]: https://github.com/kailuowang/mainecoon
