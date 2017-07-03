# Using Nexus with Docker

Nexus includes a Docker registry where private images may be hosted so that they
can be shared across teams. To configure Docker, you must first login using your
Nexus credentials:

```
$ docker login containers.example.com
```

This will ask you for your Nexus username and password.

## Publishing a Docker image

To publish a Docker image to the Nexus Docker registry you must tag the image
according to the following nomenclature:

```
containers.example.com/<group-name>/<image-name>:<tag>
```

After the image is built and tagged, you must run the following to publish it to
the registry:

```
$ docker push containers.example.com/<group-name>/<image-name>:<tag>
```

Here's a concrete example on how to build and publish an image:

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

Pulling a Docker image from the Nexus Docker registry is as simple as running

```
$ docker pull containers.example.com/<group-name>/<image-name>:<tag>
```

For example, after running the example on the previous section we could run the
following from a different workstation or server:

```
$ docker pull containers.example.com/devops/docker-nexus:3.3.2
3.3.1: Pulling from devops/docker-nexus
(...)
Digest: sha256:20878f8dda88b4ca1207d830b5445264847ccc234ded8febebb8f9e6d6483115
Status: Downloaded newer image for containers.example.com/devops/docker-nexus:3.3.2
```

The image has been pulled successfully and is now available locally.

## Pulling a Docker image from within Kubernetes

In order to pull a Docker image from a private registry from within Kubernetes
a few steps are needed. First, a secret containing the necessary Docker config
must be created. In order to do this, search your `~/.docker/config.json` for
your credentials to `containers.example.com`:

```json
{
  "auths": {
    "containers.example.com": {
      "auth": "YWRtaW46YWRtaW4xMjM="
    }
  }
}
```

Grab the base-64 encoded value in `auth` and run:

```bash
$ cat << EOF | base64
{
  "containers.example.com": {
    "username": "username",
    "password": "password",
    "email": "john.doe@example.com",
    "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
  }
}
EOF
ewogICJjb250YWluZXJzLmV4YW1wbGUuY29tIjogewogICAgInVzZXJuYW1lIjogInVzZXJuYW1lIiwKICAgICJwYXNzd29yZCI6ICJwYXNzd29yZCIsCiAgICAiZW1haWwiOiAiam9obi5kb2VAZXhhbXBsZS5jb20iLAogICAgImF1dGgiOiAiZFhObGNtNWhiV1U2Y0dGemMzZHZjbVE9IgogIH0KfQo=
```

Grab the resulting base-64 encoded value and create the following YAML file:

```yaml
apiVersion: v1
data:
  .dockercfg: ewogICJjb250YWluZXJzLmV4YW1wbGUuY29tIjogewogICAgInVzZXJuYW1lIjogInVzZXJuYW1lIiwKICAgICJwYXNzd29yZCI6ICJwYXNzd29yZCIsCiAgICAiZW1haWwiOiAiam9obi5kb2VAZXhhbXBsZS5jb20iLAogICAgImF1dGgiOiAiZFhObGNtNWhiV1U2Y0dGemMzZHZjbVE9IgogIH0KfQo=
kind: Secret
metadata:
  name: nexus-docker
type: kubernetes.io/dockercfg
```

Let Kubernetes know about your secret using `kubectl`:

```bash
$ kubectl create -f nexus-docker.yaml
secret "nexus-docker" created
```

Now, whenever you need to use a private image in a pod you just need to
reference this secret:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    -
      image: containers.example.com/private/image:1.0.0
      name: my-container
  imagePullSecrets:
    -
      name: nexus-docker
```

Kubernetes will now know how to access Nexus in order to pull your private
Docker images.
