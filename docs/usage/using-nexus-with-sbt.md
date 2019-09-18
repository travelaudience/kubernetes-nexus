# Using Nexus with sbt

Configuring `sbt` to download artifacts from Nexus instead of Maven Central
will, most of the time, not only speed up build processes by
caching commonly used dependencies but also help ensuring reproducible builds,
since one only depends on their Nexus availability and not the public repositories.

`sbt` can also be configured to upload artifacts to Nexus, enabling the management
of artifacts private to an organization.

## Downloading artifacts from Nexus

In order to enable `sbt` to download artifacts from Nexus, one is to add the
following to `build.scala`:

```scala
// This will use Nexus as a resolver.
resolvers += "My Nexus" at "https://nexus.example.com/repository/maven-public/"
// This will prevent access to Maven Central.
externalResolvers := Resolver.withDefaultResolvers(resolvers.value, mavenCentral = false)
```
if you are using `sbt > 1.x`, you need to use `combineDefaultResolvers` method instead:

```
externalResolvers := Resolver.combineDefaultResolvers(resolvers.value.toVector, false)
```


One can check if their configuration is working by deleting the `~/.ivy2/cache`
directory and running:

```shell
$ sbt run
(...)
[info] downloading https://nexus.example.com/repository/maven-public/org/scala-lang/scala-library/2.12.1/scala-library-2.12.1.jar ...
[info] 	[SUCCESSFUL ] org.scala-lang#scala-library;2.12.1!scala-library.jar (776ms)
[info] downloading https://nexus.example.com/repository/maven-public/org/scalatest/scalatest_2.12/3.0.1/scalatest_2.12-3.0.1.jar ...
[info] 	[SUCCESSFUL ] org.scalatest#scalatest_2.12;3.0.1!scalatest_2.12.jar(bundle) (889ms)
(...)
```

## Uploading artifacts to Nexus

In order to enable `sbt` to upload build artifacts to Nexus, one must configure
the `publish` task as follows:

```scala
publishTo := {
  val nexus = "https://nexus.example.com"

  if (isSnapshot.value)
    Some("snapshots" at nexus + "/repository/maven-snapshots")
  else
    Some("releases"  at nexus + "/repository/maven-releases")
}
```

For sbt 1.x one should disable gigahorse setting because of a (issue)[https://github.com/sbt/sbt/issues/3570] with GCloud hosting Nexus
```
updateOptions := updateOptions.value.withGigahorse(false),
```

A similar issue happens when assemblying a fat jar with:
```
addArtifact(artifact in (Compile, assembly), assembly)
```

Also, one must provide their Nexus credentials.
The best way is to store and load them from `~/.ivy2/.credentials`:

```scala
credentials += Credentials(Path.userHome / ".ivy2" / ".credentials")
```

`~/.ivy2/.credentials` should look like follows:

```
realm=nexus-proxy
host=nexus.example.com
user=<the-username>
password=<the-password>
```

**Attention:** If GCP IAM authentication is enabled, [username and password
**are not** the GCP organization credentials](../admin/configuring-nexus-proxy.md/#usage).

Now, running the `publish` task will look like follows:

```shell
$ sbt publish
(...)
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT.pom
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT.jar
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT-sources.jar
[info] 	published dojo-nexus-sbt_2.12 to https://nexus.example.com/repository/maven-snapshots/com/example/dojo-nexus-sbt_2.12/1.0.0-SNAPSHOT/dojo-nexus-sbt_2.12-1.0.0-SNAPSHOT-javadoc.jar
(...)
```

### Encrypting credentials

**Note**: Encrypting credentials is a security best-pratice.

Unfortunately, unlike Maven or Gradle, `sbt` doesn't seem to provide a built-in
or community-provided mechanism to encrypt credentials.
