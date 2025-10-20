# Kubernetes Setup and Usage Guide

## Table of Contents
1. [Installing Minikube in Ubuntu VM](#installing-minikube-in-ubuntu-vm)
2. [Minikube in WSL Ubuntu](#minikube-in-wsl-ubuntu)
3. [Kubernetes in Public Cloud](#kubernetes-in-public-cloud)
4. [Building All-in-one Kubernetes Distribution](#building-all-in-one-kubernetes-distribution)
5. [Demo: Installing AiO Kubernetes](#demo-installing-aio-kubernetes)
6. [Running Applications in Kubernetes](#running-applications-in-kubernetes)
7. [Working with kubectl](#working-with-kubectl)

---

## Installing Minikube in Ubuntu VM

### Prerequisites
- Virtualization software (VirtualBox, VMware, etc.)
- Ubuntu LTS Desktop VM

### Installation Steps
1. **Install Git client** in the Ubuntu VM:
sudo apt install git

2. **Clone the Git repository**:
git clone https://github.com/sandervanvugt/ckad

3. **Run the setup script**:

./minikube-docker-setup.sh

4. **Start Minikube** with Calico CNI:
minikube start --cni=calico

5. **Verify installation**:

kubectl get all

> **Note**: For virtual machine setup guidance, refer to the "Virtualization for Everyone" video course.

---

## Minikube in WSL Ubuntu

### Prerequisites
- Windows with WSL2 enabled
- Docker Desktop for Windows

### Setup Steps
1. **Install WSL2** on Windows.

2. **Install Docker Desktop** from [official Docker website](https://docs.docker.com/desktop/install/windows-install/).

3. **Verify WSL version**:

wsl -l -v

4. **Set WSL version** (if not using Ubuntu-22.04):

6. **Run the setup script** from the cloned repository.

7. **Start Minikube** with Docker driver:

minikube start --vm-driver=docker --cni=calico

7. **Remove taint from control node**:
kubectl edit node <node-name>

---

## Running Applications in Kubernetes

### Important Notes
- Many distributions provide web-based Kubernetes Dashboard
- **Do not use Dashboard during exam**:
- Might not be installed
- Provides limited functionality

### Run Your First Application

kubectl create deploy firstapp --image=nginx

---

## Working with kubectl

### Overview
`kubectl` is the core utility for interacting with Kubernetes clusters.

### Key Features
- Command-line interface for Kubernetes operations
- Enable Bash completion:
source <(kubectl completion bash)

- Use `-h` option for help at any command level
- No man pages available - `-h` covers all documentation needs

### Tip
Many `kubectl` commands include examples at the top of the `-h` output.

---

> **Note**: All Git operations reference the repository: `https://github.com/sandervanvugt/ckad`

---

## Kubernetes in Public Cloud

### Considerations
- Kubernetes is available on all major public cloud platforms
- Cloud providers handle specific functionality:
- Ingress for incoming traffic
- Network Policies for traffic limitation
- Storage allocation

### Recommendations for CKAD Preparation
- **Not recommended**: Managed public cloud solutions (sub-optimal for learning)
- **Recommended approach**: Deploy Ubuntu VM in cloud and install Minikube as described in Lesson 4.2

---
