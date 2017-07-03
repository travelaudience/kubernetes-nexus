# Using Nexus with Maven

[Maven](https://maven.apache.org/) can be configured to download artifacts from
Nexus instead of Maven Central. Most of the time this speeds up build processes
(by caching commonly used dependencies) and helps ensuring reproducible builds.

Maven can also be configured to upload artifacts to Nexus, allowing for sharing
private artifacts between teams.

## Downloading artifacts from Nexus

To make Maven use Nexus to download artifacts it's necessary to edit the global
Maven configuration (which lives in `~/.m2/settings.xml`) and make it look like
the following:

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
          <url>http://central</url>
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
          <url>http://central</url>
          <id>central</id>
          <url>http://central</url>
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

Here we define a single profile called `nexus` and configure both a `repository`
and a `pluginRepository` with the ID `central` in order to override the defaults
defined in the Maven super-POM.

You can check your configuration is working by deleting your `~/.m2/repository`
directory — making sure you have a backup — and running

```
$ mvn package
```

on any given Maven project. You should start seeing something similar to

```
$ mvn package
/Users/your-user/Workspace/dojo-nexus
[INFO] Scanning for projects...
Downloading: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-starter-parent/1.5.3.RELEASE/spring-boot-starter-parent-1.5.3.RELEASE.pom
Downloaded: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-starter-parent/1.5.3.RELEASE/spring-boot-starter-parent-1.5.3.RELEASE.pom (7.5 kB at 11 kB/s)
Downloading: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-dependencies/1.5.3.RELEASE/spring-boot-dependencies-1.5.3.RELEASE.pom
Downloaded: https://nexus.example.com/repository/maven-public/org/springframework/boot/spring-boot-dependencies/1.5.3.RELEASE/spring-boot-dependencies-1.5.3.RELEASE.pom (89 kB at 378 kB/s)
(...)
```

which means that build artifacts are being downloaded from Nexus.

## Uploading artifacts to Nexus

The upload (_deployment_) of build artifacts must be configured on a per-project
basis by editing the `distributionManagement` section of the root `pom.xml`:

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

This, in turn, requires credentials to the `nexus` repository to be specified in
the global Maven configuration (the `~/.m2/settings.xml` file we edited before):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
  <servers>
   <server>
      <id>nexus</id>
      <username>your-username</username>
      <password>your-password</password>
   </server>
  </servers>
  (...)
</settings>
```

### Encrypting credentials

It's generally—if not always—a good idea to encrypt credentials. Maven provides
an easy way to encrypt passwords used to access repositories so that they don't
end up in plain text in `settings.xml` like in the above example.

First, you need to create a _master password_. Running
`mvn --encrypt-master-password` will prompt you for one:

```
$ mvn --encrypt-master-password
Master password:
{sY/CwSgHqY8HeC4obv5H1zPBYwjt5F8k1SjeegD3/vw=}
```

Grab this value and store it in `~/.m2/security-settings.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settingsSecurity>
  <master>{sY/CwSgHqY8HeC4obv5H1zPBYwjt5F8k1SjeegD3/vw=}</master>
</settingsSecurity>
```

Now, you can start encrypting serve passwords by running

```
$ mvn --encrypt-password
Password:
{idR6S1+DYEMHFO56avrFg3NHGHOt74zdFjfKl8Cm3Bg=}
```

Replacing `your-password` in `~/.m2/settings.xml` with this value will still
allow you to deploy your artifacts to Nexus while keeping your password safe.
For further information, please refer to
[Password Encryption](https://maven.apache.org/guides/mini/guide-encryption.html)
in the Maven docs.

### Testing your configuration

You can check your configuration is working by running

```
$ mvn deploy
```

in your project. You should start seeing something similar to

```
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

which means that build artifacts are being uploaded to Nexus.
