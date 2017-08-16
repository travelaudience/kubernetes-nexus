# Using Nexus with Gradle

Configuring Gradle to download artifacts from Nexus instead of Maven Central
will, most of the time, not only speed up build processes by
caching commonly used dependencies but also help ensuring reproducible builds,
since one only depends on their Nexus availability and not the public repositories.

Gradle can also be configured to upload artifacts to Nexus, enabling the management
of artifacts private to an organization.

## Downloading artifacts from Nexus

In order to enable Gradle to to download artifacts from Nexus, one is to add the
following to `build.gradle`:

```groovy
repositories {
    maven {
        url "${nexusUrl}/repository/maven-public/"
    }
}
```

Also, one may often find `mavenCentral()` statements scattered around existing
`build.gradle` files. As a rule of the thumb, one should replace it with something
like:

```groovy
maven {
    url "${nexusUrl}/repository/maven-public/"
}
```

Finally, one must set `nexusUrl` either in the project-local `gradle.properties`
file or in the global `~/.gradle/gradle.properties` file:

```
nexusUrl=https://nexus.example.com
```

## Uploading artifacts to Nexus

In order to upload private artifacts to Nexus, one must use the `maven-publish`
plugin and configure the `publishing` extension as follows:

```groovy
apply plugin: 'maven-publish'

publishing {
    publications {
        maven(MavenPublication) {
            artifactId '<custom-artifact-id>'
            from components.java
            groupId '<custom-group-id>'
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

Then, to upload built artifacts to Nexus, one is to run:

```shell
$ gradle publish
```

**Note**: one must define their Nexus credentials in `gradle.properties`:

```
nexusUrl=https://nexus.example.com
nexusUsername=john.doe@example.com
nexusPassword=s4f3#p4ssw0rd!
```

**Attention:** If GCP IAM authentication is enabled, [username and password
**are not** the GCP organization credentials](../admin/configuring-nexus-proxy.md/#usage).

### Encrypting credentials

**Note**: Encrypting credentials is a security best-pratice.

Although Gradle doesn't provide a built-in way to encrypt credentials, there's
a [Gradle plugin](https://plugins.gradle.org/plugin/nu.studer.credentials)
which may come in handy. To use it, one must add the following to `build.gradle`:

```groovy
plugins {
    id 'nu.studer.credentials' version '1.0.3'
}
```

Then, add their Nexus password to the plugin's "keychain":

```shell
./gradlew addCredentials -PcredentialsKey="encryptedNexusPassword" -PcredentialsValue="<the-nexus-password>"
```

and adjust `build.gradle` accordingly:

```groovy
def nexusPassword = credentials.encryptedNexusPassword

publishing {
    publications {
        maven(MavenPublication) {
            artifactId '<custom-artifact-id>'
            from components.java
            groupId '<custom-group-id>'
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

**Attention:** If GCP IAM authentication is enabled, [username and password
**are not** the GCP organization credentials](../admin/configuring-nexus-proxy.md/#usage).