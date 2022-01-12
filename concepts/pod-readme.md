# Kubernetes definitions

## Container
Container is a packaged software (which includes code, runtime, system tools, system libraries and settings) created using OS-level virtualization. OS-level virtualization refers to an operating system paradigm in which the kernel allows the existence of multiple isolated user space instances.

This packaging makes containers isolated & independent of host operating system.

Containers are built on namespaces, control groups & union-capable file systems (UCFS), which makes it act like an Operating System with its own user-space but no kernel-space. Rather it uses the kernel of the host OS. [Namespaces](/concepts/definitions-readme.md#namespaces) control what a process can see i.e. Process Isolation, while control groups (Cgroups) regulate what they can use i.e. Resource Management.

These are explained in this video & blog 
* https://www.youtube.com/watch?v=el7768BNUPw 
* https://jvns.ca/blog/2016/10/10/what-even-is-a-container/
* [Union Capable File System - UCFS](https://medium.com/@knoldus/unionfs-a-file-system-of-a-container-2136cd11a779) - mounts multiple directories to a single root

## Pod
It is the smallest compute unit that can be defined, deployed and managed, encapsulating one or more containers deployed together on one host. Each pod is allocated its own internal IP address, and containers within pods can share their local storage and networking. Pods are immutable when running. Pod is like a container hosting other containers. In Kubernetes world, Pods are scaled, not the individual containers.

Each Pod has a routable IP address assigned to it, not to the containers running within it. Having a shared
network space for all containers means that the containers inside can communicate with one another over the localhost address, a feature not present in traditional Docker networking.

![Alt text](/images/pods.jpg)

> The most common use of a Pod is to run a single container. Some projects inject containers into running Pods to deliver a service. An example of this is the **Istio service mesh**, which uses this injected container as a proxy for all communication.

### ReplicaSet
The ReplicaSet maintains the desired number of copies of a Pod running within the cluster. If a Pod or the host on which it's running fails, Kubernetes launches a replacement. In all cases, Kubernetes works to maintain the desired state of the ReplicaSet.

### Deployment
A Deployment manages a ReplicaSet. Although it’s possible to launch a ReplicaSet directly or to use a ReplicationController, the use of a Deployment gives more control over the rollout strategies of the Pods that
the ReplicaSet controller manages. By defining the desired states of Pods through a Deployment, users can perform updates to the image running within the containers and maintain the ability to perform rollbacks.

### DaemonSet
A DaemonSet runs one copy of the Pod on each node in the Kubernetes cluster. This workload model provides the flexibility to run daemon processes such as log management, monitoring, storage providers, or network providers that handle Pod networking for the cluster.

### StatefulSet
A StatefulSet controller ensures that the Pods it manages have durable storage and persistent identity.
StatefulSets are appropriate for situations where Pods have a similar definition but need a unique identity,
ordered deployment and scaling, and storage that persists across Pod rescheduling.
