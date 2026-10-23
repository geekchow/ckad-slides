# 1. Containers and Images: The Building Blocks

> Series: [Working with Kubernetes](README.md) · Next: [How Kubernetes Works](02-kubernetes-architecture.md)

Kubernetes does not run your code directly. It runs **containers**, and containers start from **images**. Before you write a single Kubernetes manifest, you need a clear picture of what an image is, where it lives, and who starts it.

## The three layers: registry, runtime, engine

| Piece | What it does | Examples |
|-------|--------------|----------|
| **Registry** | Stores images and serves them to whoever pulls them | Docker Hub, Quay, GHCR, a private registry |
| **Container runtime** | The low-level program that actually starts a container process | `runc`, `crun` |
| **Container engine** | A friendly tool on top of the runtime for one machine | Docker, Podman |
| **Orchestrator** | Runs containers across many machines and keeps them running | Kubernetes |

On a laptop you use Docker or Podman to pull an image and start a container. In a cluster, Kubernetes does that job for you on every node, and it does a lot more: it restarts failed containers, spreads them over nodes and connects them to the network. That is why it is called a container *orchestrator*, not just an engine.

Public registries share images with everyone. Private registries keep images for your company only. When a registry needs a login, Kubernetes uses a `docker-registry` Secret (see [article 5](05-configmaps-and-secrets.md)).

## How images are built: layers

An image is a stack of read-only **layers**:

1. A **base image** at the bottom, such as `alpine`, `busybox` or Red Hat UBI. It is a very small Linux distribution.
2. Each step in your build adds a new layer on top (copy files, install packages and so on).

Layers are shared. If ten images use `nginx:alpine` as a base, the base layers are stored and downloaded only once. This makes images efficient to store and fast to pull.

> Base images are minimal on purpose. Tools such as `ps`, `curl` or even `bash` are often missing. If `kubectl exec` fails with `/bin/bash: not found`, try `/bin/sh`.

## Managing images locally

```bash
docker images            # list images stored on this machine
docker pull alpine       # download an image ahead of time
docker image inspect alpine
docker image prune       # remove dangling, unused images
```

When a container starts, its image must be on the node. Whether a fresh copy is pulled depends on the **image pull policy**. In Kubernetes this is `imagePullPolicy` on the container:

- `Always`: check the registry every time a container starts.
- `IfNotPresent`: pull only if the image is not on the node yet.
- `Never`: use only the local copy.

If you don't set it, Kubernetes uses `Always` for the `:latest` tag (or no tag) and `IfNotPresent` for any other tag. This is one more reason to use explicit version tags such as `nginx:1.27`.

## Three ways to create an image

| Method | When to use it |
|--------|----------------|
| `docker commit` | Quick experiments: save changes made inside a running container |
| **Dockerfile / Containerfile** | The normal way. A text file that describes the build, so it can be repeated and reviewed |
| `buildah` | Scripted builds without a daemon, often used in CI |

A Dockerfile and a Containerfile are the same thing. Podman uses the name Containerfile.

### Building from a Dockerfile

This is the Dockerfile from exam task 14 ([`exercise/ckad/task14-dockerfile`](../exercise/ckad/task14-dockerfile)):

```dockerfile
FROM nginx:alpine

# Copy static site into NGINX default directory
COPY ./index.html /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

- `FROM` picks the base image.
- `COPY` adds a layer with your file.
- `EXPOSE` documents the port. It does not publish it.
- `CMD` is the process the container runs. `daemon off` keeps nginx in the foreground. **If the main process exits, the container stops.**

Build, tag and run it:

```bash
echo "hello from my image" > index.html
docker build -f task14-dockerfile -t myapp:1.0 .
docker images | grep myapp
docker run -d -p 8080:80 myapp:1.0
curl localhost:8080
```

`-t` sets the name and tag. `-f` points to a file that is not called `Dockerfile`. The final `.` is the **build context**, which is the folder whose files `COPY` can see.

### Exporting an image to a file

Exam questions often ask you to save an image as a tar archive:

```bash
docker save myapp:1.0 -o /tmp/myapp.tar
# with Podman the commands are the same:
podman build -t myapp:1.0 .
podman save myapp:1.0 -o /tmp/myapp.tar
```

`docker load -i /tmp/myapp.tar` imports it again on another machine.

### Commit: snapshot a running container

```bash
docker run --name customweb -it nginx sh
#   inside: touch /tmp/testfile; exit
docker commit customweb nginx:custom
docker run -it nginx:custom ls -l /tmp/testfile
```

This is handy for exploring, but nobody can see how the image was made. Use a Dockerfile for anything you will keep.

## Key takeaways

- An image is a stack of layers on a small base image. A container is a running instance of an image.
- Registries store images. Runtimes start containers. Kubernetes orchestrates them across a cluster.
- Use explicit image tags and know your `imagePullPolicy`.
- `docker build -t name:tag .`, `docker save -o file.tar` and their Podman twins are must-know commands.

Next, we look at the system that runs these images at scale: [How Kubernetes Works](02-kubernetes-architecture.md).
