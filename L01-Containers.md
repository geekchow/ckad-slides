# What is a Registry?

- A registry is used as a container image store.
- Local registries can be used to provide private access to images.
- Public registries can be used to provide access to container images for a worldwide audience.
- The Docker registry is the most common registry and is often used as the default registry.

# Container Runtimes

- A container runtime is the program that starts a container.
- runc is a common container runtime.
- To manage a container that runs on a stand-alone computer, a container engine is used on top of the container runtime.
- Docker and Podman are the most common container engines for running stand-alone containers.
- To run containers in cloud, Kubernetes is used as the container engine.
- As Kubernetes is doing much more than just taking care of the running container, it is called a container orchestrator.