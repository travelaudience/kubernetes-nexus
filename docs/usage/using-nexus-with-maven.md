# Using Nexus with Maven

Configuring Maven to download artifacts from Nexus instead of Maven Central
will, most of the time, not only speed up build processes by
caching commonly used dependencies but also help ensuring reproducible builds,
since one only depends on their Nexus availability and not the public repositories.

Maven can also be configured to upload artifacts to Nexus, enabling the management
of artifacts private to an organization.

## Configuring credentials

To be able to upload/download artifacts, first it is required to specify the credentials to the `nexus` repository in the global Maven configuration (the `~/.m2/settings.xml`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
  <servers>
   <server>
      <id>nexus</id>
      <username>the-username</username>
      <password>the-password</password>
   </server>
  </servers>
  (...)
</settings>
```

**Attention:** If GCP IAM authentication is enabled, username and password
**are not the GCP organization credentials** but are instead the [credentials obtained with Nexus cli](../admin/configuring-nexus-proxy.md#using-command-line-tools).

### Encrypting credentials

**Note**: Encrypting credentials is a security best-pratice.

Maven provides an easy way to encrypt passwords for securely storing the
credentials needed to access private repositories. In order to do so, one must
create a _master password_ as follows:

```shell
$ mvn --encrypt-master-password
Master password:
{sY/CwSgHqY8HeC4obv5H1zPBYwjt5F8k1SjeegD3/vw=}
```

One is to copy the resulting hash value and store it in `~/.m2/security-settings.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settingsSecurity>
  <master>{sY/CwSgHqY8HeC4obv5H1zPBYwjt5F8k1SjeegD3/vw=}</master>
</settingsSecurity>
```

Now, one can start encrypting passwords by running:

```
$ mvn --encrypt-password
Password:
{idR6S1+DYEMHFO56avrFg3NHGHOt74zdFjfKl8Cm3Bg=}
```

Replacing `the-password` in `~/.m2/settings.xml` with this value will still
allow one to deploy private artifacts to Nexus while keeping one's credentials
secure.
For further information, please refer to
[Password Encryption](https://maven.apache.org/guides/mini/guide-encryption.html).

**Attention:** If GCP IAM authentication is enabled, username and password
**are not the GCP organization credentials** but are instead the [credentials obtained with Nexus cli](../admin/configuring-nexus-proxy.md#using-command-line-tools).

## Downloading artifacts from Nexus

In order to enable Maven to download artifacts from Nexus, one is to edit the
the user's global Maven configuration (which lives in `~/.m2/settings.xml`) as
follows:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.example.com/repository/maven-public/</url>
    </mirror>
  </mirrors>
  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://central</url>
          <releases>
            <enabled>true</enabled>
          </releases>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        </repository>
      </repositories>
     <pluginRepositories>
        <pluginRepository>
          <id>central</id>
          <url>https://central</url>
          <releases>
            <enabled>true</enabled>
          </releases>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>
</settings>
```

Above, a single profile called `nexus` includes both a `repository` and a
`pluginRepository` with the ID `central` in order to override the defaults
defined in the Maven super-POM.

One can check if their configuration is working by deleting the `~/.m2/repository`
directory and running:

```shell
$ mvn package
```

on any given Maven project. Public dependencies should now be downloaded from Nexus
instead of the public repositories:

```
$ mvn compile
/Users/the-user/Workspace/dojo-nexus
[INFO] Scanning for projects...
Downloading: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-starter-parent/1.5.3.RELEASE/spring-boot-starter-parent-1.5.3.RELEASE.pom
Downloaded: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-starter-parent/1.5.3.RELEASE/spring-boot-starter-parent-1.5.3.RELEASE.pom (7.5 kB at 11 kB/s)
Downloading: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-dependencies/1.5.3.RELEASE/spring-boot-dependencies-1.5.3.RELEASE.pom
Downloaded: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-dependencies/1.5.3.RELEASE/spring-boot-dependencies-1.5.3.RELEASE.pom (89 kB at 378 kB/s)
(...)
```

## Uploading artifacts to Nexus

In order to upload private artifacts to Nexus, one must configure the
`distributionManagement` section of the root `pom.xml` as follows:

```xml
<distributionManagement>
    <repository>
        <id>nexus</id>
        <name>Releases</name>
        <url>https://nexus.example.com/repository/maven-releases</url>
    </repository>
    <snapshotRepository>
        <id>nexus</id>
        <name>Snapshot</name>
        <url>https://nexus.example.com/repository/maven-snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

After one is done with Maven configuration, publishing private artifacts should
be as easy as running:

```
$ mvn deploy

(...)
[INFO] --- maven-deploy-plugin:2.8.2:deploy (default-deploy) @ dojo-nexus ---
Downloading: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/maven-metadata.xml
Downloaded: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/maven-metadata.xml (781 B at 1.8 kB/s)
Downloading: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/maven-metadata.xml
Downloaded: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/maven-metadata.xml (781 B at 8.1 kB/s)
Uploading: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/dojo-nexus-2.0.0-20170610.093044-6.jar
Uploaded: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/dojo-nexus-2.0.0-20170610.093044-6.jar (6.6 MB at 5.7 MB/s)
Uploading: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/dojo-nexus-2.0.0-20170610.093044-6.pom
Uploaded: https://nexus.example.com/repository/maven-snapshots/com/example/dojo/dojo-nexus/2.0.0-SNAPSHOT/dojo-nexus-2.0.0-20170610.093044-6.pom (1.7 kB at 5.3 kB/s)
(...)
```
