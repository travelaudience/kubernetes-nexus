# Using Nexus with Docker

Nexus includes a Docker registry where private images may be hosted so that they
can be shared across teams. To configure Docker, one must first login using one's
Nexus credentials:

```
$ docker login containers.example.com
```

**ATTENTION:** If GCP IAM authentication is enabled, [username and password
**are not** the GCP organization credentials](../admin/configuring-nexus-proxy.md/#usage).

## Publishing a Docker image

To publish a Docker image to the Nexus Docker registry one must tag the image
as follows:

```
containers.example.com/<group-name>/<image-name>:<tag>
```

After the image is built and tagged, one must run the following to publish it to
the Nexus registry:

```
$ docker push containers.example.com/<group-name>/<image-name>:<tag>
```

Here's a concrete example of how to build and publish a private container image:

```
$ docker build -t containers.example.com/devops/docker-nexus:3.3.2 .
(...)
Successfully built 5f1d02765ee9
Successfully tagged containers.example.com/devops/docker-nexus:3.3.2
$ docker push containers.example.com/devops/docker-nexus:3.3.2
The push refers to a repository [containers.example.com/devops/docker-nexus]
(...)
3.3.2: digest: sha256:20878f8dda88b4ca1207d830b5445264847ccc234ded8febebb8f9e6d6483115 size: 2817
```

## Pulling a Docker image

Pulling a Docker image from the Nexus Docker registry is as simple as running:

```
$ docker pull containers.example.com/<group-name>/<image-name>:<tag>
```

Here's a concrete example of how to pull a private container image:

```
$ docker pull containers.example.com/devops/docker-nexus:3.3.2
3.3.1: Pulling from devops/docker-nexus
(...)
Digest: sha256:20878f8dda88b4ca1207d830b5445264847ccc234ded8febebb8f9e6d6483115
Status: Downloaded newer image for containers.example.com/devops/docker-nexus:3.3.2
```

The image has been pulled successfully and is now available locally.

## Pulling a Docker image from within Kubernetes

For Kubernetes to be able to pull a private container image, a secret containing
the necessary Docker configuration must be created. Below one can see 2 different ways of doing the first part which is getting the **auth** field.

### Linux

```shell
$ cat ~/.docker/config.json
```

The result should look as below:

```json
{
  "auths": {
    "containers.example.com": {
      "auth": "YWRtaW46YWRtaW4xMjM="
    }
  }
}
```

### OSX

Since OSX uses keychain one cannot use the info in the `config.json` file.
Instead the **auth** content can be generated with the command below:
```bash
echo -n "username:password" | base64
```

Example:
```bash
echo -n "admin:admin123" | base64
YWRtaW46YWRtaW4xMjM=
```

## Using secret type `kubernetes.io/dockercfg`

One is to copy the base-64 encoded value of key `auth` and run:

```bash
$ cat << EOF | base64
{
  "auths": {
    "containers.example.com": {
      "email": "john.doe@example.com",
      "auth": "YWRtaW46YWRtaW4xMjM="
    }
  }
}
EOF
ewogICJjb250YWluZXJzLmV4YW1wbGUuY29tIjogewogICAgInVzZXJuYW1lIjogInVzZXJuYW1lIiwKICAgICJwYXNzd29yZCI6ICJwYXNzd29yZCIsCiAgICAiZW1haWwiOiAiam9obi5kb2VAZXhhbXBsZS5jb20iLAogICAgImF1dGgiOiAiZFhObGNtNWhiV1U2Y0dGemMzZHZjbVE9IgogIH0KfQo=
```

**ATTENTION**: one must replace the keys and values above according to one's
environment, i.e. replace `containers.example.com` with the right hostname.

Now, one is to copy the resulting base-64 encoded value and create the following
Kubernetes secret descriptor:

```yaml
apiVersion: v1
data:
  .dockercfg: ewogICJjb250YWluZXJzLmV4YW1wbGUuY29tIjogewogICAgInVzZXJuYW1lIjogInVzZXJuYW1lIiwKICAgICJwYXNzd29yZCI6ICJwYXNzd29yZCIsCiAgICAiZW1haWwiOiAiam9obi5kb2VAZXhhbXBsZS5jb20iLAogICAgImF1dGgiOiAiZFhObGNtNWhiV1U2Y0dGemMzZHZjbVE9IgogIH0KfQo=
kind: Secret
metadata:
  name: nexus-docker
type: kubernetes.io/dockercfg
```

Store the secret:

```bash
$ kubectl create -f nexus-docker.yaml
secret "nexus-docker" created
```

Now, whenever one need to use a private image in a pod, one just need to
reference the newly created secret:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    -
      image: containers.example.com/devops/docker-nexus:3.3.2
      name: my-container
  imagePullSecrets:
    -
      name: nexus-docker
```

## Using secret type `kubernetes.io/dockerconfigjson`

One is to copy the base-64 encoded value of key `auth` and run:

```bash
$ cat << EOF | base64
{
  "auths": {
    "containers.example.com": {
      "email": "john.doe@example.com",
      "auth": "YWRtaW46YWRtaW4xMjM="
    }
  }
}
EOF
ewogICJhdXRocyI6IHsKICAgICJjb250YWluZXJzLmV4YW1wbGUuY29tIjogewogICAgICAiZW1haWwiOiAiam9obi5kb2VAZXhhbXBsZS5jb20iLAogICAgICAiYXV0aCI6ICJZV1J0YVc0NllXUnRhVzR4TWpNPSIKICAgIH0KICB9Cn0K
```

**ATTENTION**: one must replace the keys and values above according to one's
environment, i.e. replace `containers.example.com` with the right hostname.

Now, one is to copy the resulting base-64 encoded value and create the following
Kubernetes secret descriptor:

```yaml
apiVersion: v1
data:
  .dockerconfigjson: ewogICJhdXRocyI6IHsKICAgICJjb250YWluZXJzLmV4YW1wbGUuY29tIjogewogICAgICAiZW1haWwiOiAiam9obi5kb2VAZXhhbXBsZS5jb20iLAogICAgICAiYXV0aCI6ICJZV1J0YVc0NllXUnRhVzR4TWpNPSIKICAgIH0KICB9Cn0K
kind: Secret
metadata:
  name: nexus-docker
type: kubernetes.io/dockerconfigjson
```

Store the secret:

```bash
$ kubectl create -f nexus-docker.yaml
secret "nexus-docker" created
```

Now, whenever one need to use a private image in a pod, one just need to
reference the newly created secret:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    -
      image: containers.example.com/devops/docker-nexus:3.3.2
      name: my-container
  imagePullSecrets:
    -
      name: nexus-docker
```
