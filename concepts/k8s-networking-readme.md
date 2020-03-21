# Kubernetes networking
Kubernetes networking builds on top of the Docker and [Netfilter](/concepts/definitions-readme.md#netfilter) constructs to tie multiple components together into applications. Kubernetes resources have specific names and capabilities, and we need to understand those before exploring their inner workings.

* [Container, Pod & other k8s definitions](/concepts/pod-readme.md)

## Pod Networking
The Pod is the smallest unit in Kubernetes, so it is essential to first understand Kubernetes networking in the context of communication between Pods. Because a Pod can hold more than one container, we can start with a look at how communication happens between containers in a Pod. Although Kubernetes can use Docker for the underlying container runtime, its approach to networking differs slightly and imposes some basic principles:

* Any Pod can communicate with any other Pod without the use of network address translation (NAT). To facilitate this, Kubernetes assigns each Pod an IP address that is routable within the cluster.
* A node can communicate with a Pod without the use of NAT.
* A Pod's awareness of its address is the same as how other resources see the address. The host's address doesn't mask it.

These principles give a unique and first-class identity to every Pod in the cluster. Because of this, the networking model is more straightforward and does not need to include port mapping for the running container workloads.

* [Docker vs Kubernetes networking](/concepts/docker-k8s-networking-readme.md)
    * Explained in a [youtube video](https://youtu.be/GXq3FS8M_kw?t=1345)

## Pause container

