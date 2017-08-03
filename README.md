# kubernetes-nexus

Nexus Repository Manager OSS (3.3.2) on top of Kubernetes.

## Table of Contents

* [Pre-Requisites](#pre-requisites)
* [Deployment](#deployment)
  * [Deploying Nexus](#deploying-nexus)
  * [Securing the Deployment with HTTPS](#securing-the-deployment-with-https)
  * [Configuring Nexus](#configuring-nexus)
  * [Configuring Backup Retention](#configuring-backup-retention)
* [Using Nexus](#usage)
  *  [Docker](#usage-docker)
  *  [Maven](#usage-maven)
  *  [Gradle](#usage-gradle)
  *  [sbt](#usage-sbt)
  *  [Python](#usage-python)
* [Backup and Restore](#backup-and-restore)
  * [Backup](#backup)
  * [Restore](#restore)

## Pre-Requisites

* A working Kubernetes cluster (v1.7.1 or newer) with Cloud Storage read-write
permissions enabled (`https://www.googleapis.com/auth/devstorage.read_write` scope)
* A working installation of `kubectl` configured to access the
  cluster.
* A working installation of `gcloud` configured to access the Google Cloud
  Platform project.
* A global static IPv4 address (e.g., `your-static-ip-name`).
* A DNS A record `nexus.example.com` pointing to this IPv4 address.
* A DNS CNAME record `containers.example.com` pointing to
  `nexus.example.com`.
* A [Google Cloud Storage](https://cloud.google.com/storage/) bucket (e.g.,
  `nexus-backup`).

## Deployment

### Deploying Nexus

The first thing you should do after deploying Nexus is to change the default
admin credentials (`admin:admin123`) to something secure. Thus, and since the
`nexus-backup` container uses these credentials to access the Nexus API, you
should update
[`nexus-secret.yaml`](kubernetes/nexus-secret.yaml)
to reflect this (future) change:

```bash
$ cd nexus/
$ NEXUS_CREDENTIALS=$(echo -n '<new-username>:<new-password>' | base64)
$ NEXUS_AUTH=$(echo -n "Basic ${NEXUS_CREDENTIALS}" | base64)
$ sed -i.bkp "s/QmFzaWMgWVdSdGFXNDZZV1J0YVc0eE1qTT0=/${NEXUS_AUTH}/" nexus-secret.yaml
```

After updating `nexus-secret.yaml`, deploying Nexus is a matter of running

```bash
$ kubectl create -f nexus-configmap.yaml
$ kubectl create -f nexus-secret.yaml
$ kubectl create -f nexus-statefulset.yaml
$ kubectl create -f nexus-docker-svc.yaml
$ kubectl create -f nexus-http-svc.yaml
$ kubectl create -f nexus-ingress.yaml
```

You should allow for 5 to 10 minutes for GCLB to update. Nexus should then
become available over HTTP at http://nexus.example.com.

### Securing the Deployment with HTTPS

In order to secure the deployment one must configure HTTPS access. The easiest
and cheapest way to obtain a trusted TLS certicate is using
[Let's Encrypt](https://letsencrypt.org/), and the easiest way to automate the
process of obtaining and renewing certificates from Let's Encrypt is by using
[`kube-lego`](https://github.com/jetstack/kube-lego):

```bash
$ cd kube-lego/
$ kubectl create -f kube-lego-namespace.yaml
$ kubectl create -f kube-lego-configmap.yaml
$ kubectl create -f kube-lego-deployment.yaml
```

As soon as it starts, `kube-lego` will start monitoring _Ingress_ resources and
requesting certificates from Let's Encrypt. Please note that Let's Encrypt must
be able to reach port `80` on domains for which certificates are requested, and
thus the value of the `kubernetes.io/ingress.allow-http` annotation on
[`nexus-ingress.yaml`](kubernetes/nexus-ingress.yaml)
must be set to `"true"`.

If everything goes well, after a while you will
be able to access https://nexus.example.com securely.

### Configuring Nexus

Please head over to
[`docs/admin/configuring-nexus.md`](docs/admin/configuring-nexus.md)
for information on how to configure Nexus.

### Configuring Backup Retention

**Attention**: As mentioned in the pre-requisites, the GKE cluster needs read-write
permissions on GCP Cloud Storage in order to upload backups.

The backup procedure uses Google Cloud Storage to save backups. In order to
configure a backup retetion policy, head over to the `backup-lifecycle`
directory and install one of the available policies by running:

```bash
$ ./gsutil-lifecycle-set <policy-file> <bucket-name>
```

Google Cloud Storage will then automatically purge backups older than the number
of days specified.

<a id="usage">

## Using Nexus

Below are linked detailed instructions on how to configure a bunch of tools
in order to download and upload artifacts from and to Nexus.

<a id="usage-docker">

### Docker

Please, read [Using Nexus with Docker](docs/usage/using-nexus-with-docker.md).

<a id="usage-maven">

### Maven

Please, read [Using Nexus with Maven](docs/usage/using-nexus-with-maven.md).

<a id="usage-gradle">

### Gradle

Please, read [Using Nexus with Gradle](docs/usage/using-nexus-with-gradle.md).

<a id="usage-sbt">

### sbt

Please, read [Using Nexus with sbt](docs/usage/using-nexus-with-sbt.md).

<a id="usage-python">

### Python

Please, read [Using Nexus with Python](docs/usage/using-nexus-with-python.md).

## Backup and Restore

**Attention**: As mentioned in the pre-requisites, the GKE cluster needs read-write
permissions on GCP Cloud Storage in order to upload backups.

### Backup

Nexus has a built-in
[_Export configuration & metadata for backup_](http://books.sonatype.com/nexus-book/3.3/reference/backup.html#backup-task)
task which we use to backup configuration and metadata. However, the generated
backup doesn't include
[blob stores](http://books.sonatype.com/nexus-book/3.3/reference/admin.html#admin-repository-blobstores),
rendering it semi-useless in a disaster recovery scenario. It is thus of the
utmost importance to backup blob stores separately and at roughly the same time
this task runs in order to achieve consistent backups.

This is the role of the
[`nexus-backup`](https://github.com/travelaudience/docker-nexus-backup)
container — a container with a script which makes backups of the `default`
blob store and then uploads them on a Google Cloud Storage bucket alongside the
data generated by Nexus built-in backup task. These backups are triggered by
a Nexus task of type _Execute script_ (to be configured as described in
[`docs/admin/configuring-nexus.md`](docs/admin/configuring-nexus.md)).
The script reacts to changes made to

```
/nexus-data/backup/.backup
```

and initiates the backup process:

1. Check if Nexus can be reached. Proceed only if Nexus is reached successfully.
1. Retrieve the current timestamp in order to tag the backup appropriately.
1. Make sure that the scripts used to start/stop repositories are installed.
1. Stop all the repositories so that no writes are made during the backup.
1. Give Nexus a _grace period_ for the configuration and metadata backup task to
   complete.
1. Archive the `default` blob store using `tar`,
   [_streaming_](https://cloud.google.com/storage/docs/streaming) the resulting
   file to the target bucket.
1. Archive the backup files generated by Nexus using `tar`, _streaming_ the
   resulting file to the target bucket.
1. Cleanup all leftover `*.bak` files in `/nexus-data/backup`.
1. Start all the repositories so that Nexus can become available again.

It is advisable to configure this backup procedure to run at off-peak hours, as
described in the aforementioned document.

### Restore

In a disaster recovery scenario, the latest backup made by the `nexus-backup`
container should be restored. In order to achieve this, Nexus must be stopped.
The procedure goes as follows:

```
$ kubectl exec -i -t nexus-0 /bin/sh # Enter the container.
$ mv /etc/service/nexus/ /nexus-service/         # Prevent `runsvdir` from respawning Nexus after the next step is performed.
$ pkill java                                     # Ask for Nexus to terminate gracefully.
```

At this point, Nexus is stopped but the container is still running, giving us a
chance to perform the restore procedure. It should go as follows:

1. Retrieve from GCS the `blobstore.tar` and `databases.tar` files
   from the last known good backup.
1. Remove everything _under_ `/nexus-data/backup/` and
   `/nexus-data/blob/default/`.
1. Untar `blobstore.tar` into `/nexus-data/blob/default/`, stripping paths if
   necessary.
1. Untar `databases.tar` into `/nexus-data/backup/`, stripping paths if
   necessary.

At this point the backup is ready to be restored by Nexus, which will happen
automatically once the service is started:

```
$ mv /nexus-service/ /etc/service/nexus/         # Make `runsvdir` start Nexus again.
$ exit                                           # Bye!
```

If you watch the `nexus` container logs as it boots you should see an indication
that a restore procedure is in place. After a few minutes access should be
restored with the last known good backup in place.