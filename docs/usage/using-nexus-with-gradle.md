# Using Nexus with Gradle

[Gradle](https://gradle.org) can be configured to download artifacts from Nexus
instead of Maven Central. Most of the time this speeds up build processes (by
caching commonly used dependencies) and helps ensuring reproducible builds.

Gradle can also be configured to upload artifacts to Nexus, allowing for sharing
private artifacts between teams.

## Downloading artifacts from Nexus

To make Gradle use Nexus to download artifacts, add this to your `build.gradle`:

```groovy
repositories {
    maven {
        url "${nexusUrl}/repository/maven-public/"
    }
}
```

Also, you may often find `mavenCentral()` statements scattered around existing
`build.gradle` files. As a rule of the thumb you can replace every occurrence of
this with

```groovy
maven {
    url "${nexusUrl}/repository/maven-public/"
}
```

Finally, you must define `nexusUrl` either in your project-local
`gradle.properties` file or in the global `~/.gradle/gradle.properties` file:

```
nexusUrl=https://nexus.example.com
```

## Uploading artifacts to Nexus

To upload build artifacts to Nexus you must use the `maven-publish` plugin and
configure the `publishing` extension according to this:

```groovy
apply plugin: 'maven-publish'

publishing {
    publications {
        maven(MavenPublication) {
            artifactId '<your-artifact-id>'
            from components.java
            groupId '<your-group-id>'
            version project.version
        }
    }
    repositories {
        maven {
            credentials {
                username nexusUsername
                password nexusPassword
            }
            if (!project.version.endsWith('-SNAPSHOT')) {
                url "${nexusUrl}/repository/maven-releases"
            } else {
                url "${nexusUrl}/repository/maven-snapshots"
            }
        }
    }
}
```

Then, to upload your build artifacts to Nexus, just run:

```
$ gradle publish
```

Please note that this requires that you define your Nexus credentials in
`gradle.properties`:

```
nexusUrl=https://nexus.example.com
nexusUsername=john.doe@example.com
nexusPassword=s4f3#p4ssw0rd!
```

### Encrypting credentials

It's generally—if not always—a good idea to encrypt credentials. Although Gradle
doesn't provide a built-in way to encrypt credentials there's a
[Gradle plugin](https://plugins.gradle.org/plugin/nu.studer.credentials)
which may come in handy. To use it, add this to your `build.gradle`:

```groovy
plugins {
    id 'nu.studer.credentials' version '1.0.3'
}
```

Then, add your Nexus password to the plugin's "keychain"

```
./gradlew addCredentials -PcredentialsKey="encryptedNexusPassword" -PcredentialsValue="<your-nexus-password>"
```

and adjust your `build.gradle` accordingly:

```groovy
def nexusPassword = credentials.encryptedNexusPassword

publishing {
    publications {
        maven(MavenPublication) {
            artifactId '<your-artifact-id>'
            from components.java
            groupId '<your-group-id>'
            version project.version
        }
    }
    repositories {
        maven {
            credentials {
                username nexusUsername
                password nexusPassword
            }
            if (!project.version.endsWith('-SNAPSHOT')) {
                url "${nexusUrl}/repository/maven-releases"
            } else {
                url "${nexusUrl}/repository/maven-snapshots"
            }
        }
    }
}
```
