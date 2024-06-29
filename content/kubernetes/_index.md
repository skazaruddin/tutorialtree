+++
categories = "kubernetes"
disableToc = true
title = "Kubernetes"
weight = 1
description = "Get started with Kubernetes, the world’s most popular container orchestration tool."

+++

Get started with Kubernetes, the world’s most popular container orchestration tool.

Kubernetes is an open-source container orchestration tool used for automating the deployment, scaling, and management of containerized applications. Developed by Google and handed over to the Cloud Native Computing Foundation (CNCF), Kubernetes has become the standard for container orchestration.

## Kubernetes Architecture

Kubernetes architecture consists of two main components: the Control Plane and the Data Plane.

### Control Plane

The control plane is responsible for maintaining the desired state of the cluster, such as which applications are running and on which nodes. It consists of the following components:

- **etcd**: A distributed key-value store for configuration data.
- **API Server**: A REST API Server that exposes the Kubernetes API spec, acting as a front-end for the control plane.
- **Controller Manager**: Manages the state of the cluster, such as replicating pods or handling node failures.
- **Cloud Controller Manager**: Manages the state of the cluster and implements the cloud provider API for cloud interaction.
- **Scheduler**: Decides on which node a pod should run.

### Data Plane

The data plane is responsible for running the containers. It consists of the following components:

- **Kubelet**: An agent that runs on each node, responsible for starting and stopping containers.
- **Kube-Proxy**: A network proxy that runs on each node, responsible for forwarding network traffic to the correct pods.
- **Container Runtime**: The low-level software that actually starts and stops containers, such as Docker or CRI-O.
