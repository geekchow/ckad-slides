L02-Managing Container Images

# Image Architecture

- While creating application images, common system images are often used as the foundation.
- This can be seen in Dockerfile, where the FROM statement is used to refer to the system image that is to be used.
- The modifications made by adding the specific applications are stored in separate layers.
- This makes working with container images efficient.

# Managing Container Images

- Before starting a container, required images are pulled and stored.
- Images can be pre-fetched using the docker pull command.
- Use docker images for a list of currently stored images.
- According to the image pull policy, a new image may automatically be pulled if it's available.
  - The default image pull policy in Docker is set to always.
  - In Kubernetes, it can be set to newer.
- To remove unused images, use docker image prune.

# Demo: Managing Images

- docker images
- docker pull alpine
- docker images
- docker image prune

# Image Creation Options

- Containers have become the standard for distributing applications.
- Creating custom images is easy.
- **docker commit** allows you to create a custom image by saving changes made to a running container.
- **Dockerfile** A.K.A. Containerfile builds a custom image based on components defined in the file.
- **buildah** is an advanced solution to build custom images in a scripted way.

# Base Images

- To create custom images, base images are normally used.
- A base image is a minimized image that can be used as the foundation for building your own images.
- Often, the base image is a fully functional, minimized Linux distribution.
- However, it should not be expected that all tools are always available.
  - Common tools like ps are often missing as they are not needed to run the main container application.
- Common base images are:
  - Busybox
  - Alpine
  - Red Hat UBI (Universal Base Image)

# Using Dockerfile

- Dockerfile can be provided by application developers.
- In Podman environments, Dockerfile is referred to as Containerfile, there are no functional differences.
- It's also relatively easy to write your own.
- To build an image from a Dockerfile, use `docker build -t imagename .`
- In this command, -t (tag) specifies the name of the image you want to create.
- . refers to the current directory as the directory where the Dockerfile is found.


# Demo: Build an Image from Dockerfile

- cd ckad/dockerfile
- cat Dockerfile
- docker build -t myapp .
- docker images
- docker image inspect myapp
- docker run myapp


# Container Modification and Commit Example

- docker run --name customweb -it nginx sh
  - touch /tmp/testfile
  - exit
- docker commit customweb nginx:custom
- docker images
- docker run -it nginx:custom ls -l /tmp/testfile