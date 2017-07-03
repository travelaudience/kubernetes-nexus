# Using Nexus with SBT

[SBT](http://www.scala-sbt.org/) can be configured to download artifacts from
Nexus instead of Maven Central. Most of the time this speeds up build processes
(by caching commonly used dependencies) and helps ensuring reproducible builds.

SBT can also be configured to upload artifacts to Nexus, allowing for sharing
private artifacts between teams.

## Downloading artifacts from Nexus

To make SBT use Nexus to download artifacts, add this to your `build.scala`:

```scala
// This will use Nexus as a resolver.
resolvers += "My Nexus" at "https://nexus.example.com/repository/maven-public/"
// This will prevent access to Maven Central.
externalResolvers := Resolver.withDefaultResolvers(resolvers.value, mavenCentral = false)
```

You can check that your configuration is working by deleting the `~/.ivy2/cache`
directory — making sure you have a backup — and running

```
$ sbt run
(...)
[info] downloading https://nexus.example.com/repository/maven-public/org/scala-lang/scala-library/2.12.1/scala-library-2.12.1.jar ...
[info] 	[SUCCESSFUL ] org.scala-lang#scala-library;2.12.1!scala-library.jar (776ms)
[info] downloading https://nexus.example.com/repository/maven-public/org/scalatest/scalatest_2.12/3.0.1/scalatest_2.12-3.0.1.jar ...
[info] 	[SUCCESSFUL ] org.scalatest#scalatest_2.12;3.0.1!scalatest_2.12.jar(bundle) (889ms)
(...)
```

## Uploading artifacts to Nexus

To upload build artifacts to Nexus you must configure the `publish` task:

```scala
publishTo := {
  val nexus = "https://nexus.example.com"

  if (isSnapshot.value)
    Some("snapshots" at nexus + "/repository/maven-snapshots") 
  else
    Some("releases"  at nexus + "/repository/maven-releases")
}
```

Also, you must provide your Nexus credentials. The best way is to load them from
`~/.ivy2/.credentials`:

```scala
credentials += Credentials(Path.userHome / ".ivy2" / ".credentials")
```

Then, create `~/.ivy2/.credentials` and add the following lines:

```
realm=Sonatype Nexus Repository Manager
host=nexus.example.com
user=<your-username>
password=<your-password>
```

Now, running the `publish` task will output something along the lines of:

```
$ sbt publish
(...)
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT.pom
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT.jar
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT-sources.jar
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT-javadoc.jar
(...)
```

### Encrypting credentials

It's generally—if not always—a good idea to encrypt credentials. However, unlike
Maven or Gradle, SBT doesn't provide a built-in or easy way to encrypt
repository credentials.
