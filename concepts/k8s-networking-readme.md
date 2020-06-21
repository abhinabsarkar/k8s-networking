# Kubernetes networking
Kubernetes networking builds on top of the Docker and [Netfilter](/concepts/definitions-readme.md#netfilter) constructs to tie multiple components together into applications. Kubernetes resources have specific names and capabilities, and we need to understand those before exploring their inner workings.

* [Container, Pod & other k8s definitions](/concepts/pod-readme.md)

## Pod Networking
The Pod is the smallest unit in Kubernetes, so it is essential to first understand Kubernetes networking in the context of communication between Pods. Because a Pod can hold more than one container, we can start with a look at how communication happens between containers in a Pod. Although Kubernetes can use Docker for the underlying container runtime, its approach to networking differs slightly and imposes some basic principles:

* Any Pod can communicate with any other Pod without the use of network address translation (NAT). To facilitate this, Kubernetes assigns each Pod an IP address that is routable within the cluster.
* A node can communicate with a Pod without the use of NAT.
* A Pod's awareness of its address is the same as how other resources see the address. The host's address doesn't mask it.

K8s natively supports multi-host networking in which pods are able to communicate with each other by default, regardless of which host they live in. Kubernetes does not provide an implementation of this model by default, rather it relies on third-party tools *"network plugins"* (eg. kubenet, azure in AKS) that comply with the following requirements: all containers are able to communicate with each other without NAT; nodes are able to communicate with containers without NAT; and a container’s IP address is the same from inside and outside the container.

These principles give a unique and first-class identity to every Pod in the cluster. Because of this, the networking model is more straightforward and does not need to include port mapping for the running container workloads.

* [Docker vs Kubernetes networking](/concepts/docker-k8s-networking-readme.md)
    * Explained in a [youtube video](https://youtu.be/GXq3FS8M_kw?t=1345)

## Pause container
Pause container has the network namespace which is used by the other containers running in a pod. Pod contains a pause container, along with app container(s) which use the pause container for their networking.

### Demo
Launch a pod running nginx & inspect the docker container running within the pod
```bash
# Create a pod running nginx
> kubectl run nginx --image=nginx
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created

# Get the pod running
> kubectl get pods -o wide | grep nginx
nginx-7bb7cd8db5-dfttx   1/1     Running   0    22s   10.1.0.247   docker-desktop   <none>     <none>

# Get the docker containers running
> docker ps | grep nginx
94c2fe8a82fb        nginx                  "nginx -g 'daemon of…"   24 seconds ago      Up 23 seconds                                                  k8s_nginx_nginx-7bb7cd8db5-dfttx_default_3da5af5a-f28d-4b79-b3e6-4f70a1d08049_0
42fef8893baf        k8s.gcr.io/pause:3.1   "/pause"                 27 seconds ago      Up 25 seconds                                                  k8s_POD_nginx-7bb7cd8db5-dfttx_default_3da5af5a-f28d-4b79-b3e6-4f70a1d08049_0
```
When we do so, we see that the container does not have the network settings provided to it. The pause  container which runs as part of the Pod is the one which gives the networking constructs to the Pod.
```bash
# Check the docker container network settings for the nginx pod. Network settings will be empty
> docker inspect 94c2fe8a82fb --format="{{json .NetworkSettings}}"
{"Bridge":"","SandboxID":"","HairpinMode":false,"LinkLocalIPv6Address":"","LinkLocalIPv6PrefixLen":0,"Ports":{},"SandboxKey":"","SecondaryIPAddresses":null,"SecondaryIPv6Addresses":null,"EndpointID":"","Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"IPAddress":"","IPPrefixLen":0,"IPv6Gateway":"","MacAddress":"","Networks":{}}

# Check the docker container network settings for the pause container. Network settings will be populated
> docker inspect 42fef8893baf --format="{{json .NetworkSettings}}"
{"Bridge":"","SandboxID":"ecc0a63a1335339eaa1efaf4f62d44515ee3faa6e1654be205c45cfd1602c7dc","HairpinMode":false,"LinkLocalIPv6Address":"","LinkLocalIPv6PrefixLen":0,"Ports":{},"SandboxKey":"/var/run/docker/netns/ecc0a63a1335","SecondaryIPAddresses":null,"SecondaryIPv6Addresses":null,"EndpointID":"","Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"IPAddress":"","IPPrefixLen":0,"IPv6Gateway":"","MacAddress":"","Networks":{"none":{"IPAMConfig":null,"Links":null,"Aliases":null,"NetworkID":"4033f9178e0efe9894bda56d490213781c5f45040ed1893d7bb817d84d2d92d9","EndpointID":"1326dc3842081690773c4e3f79ba2f88af8b40c27b743128ad716cbcbc04785f","Gateway":"","IPAddress":"","IPPrefixLen":0,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"","DriverOpts":null}}}
```

```bash
# Delete the deployment to clean up
> kubectl delete deployments nginx
deployment.extensions "nginx" deleted
```

## Intra-pod communication
Kubernetes follows the IP-per-Pod model where it assigns a routable IP address to the Pod. The containers within the Pod share the same network space and communicate with one another over localhost. Like processes running on a host, two containers cannot each use the same network port, but it can be worked around by changing the manifest.

## Inter-pod communication
Because it assigns routable IP addresses to each Pod, and because it requires that all resources see the address of a Pod the same way, Kubernetes assumes that all Pods communicate with one another via their assigned addresses. Doing so removes the need for an external service discovery mechanism.

## Kubernetes Service

* [Services](/concepts/service-readme.md)

## Kubelet
It is the primary “node agent” that runs on each node. They ensure the containers described in the PodSpecs (YAML file describing the Pod) are running and healthy.

## Kube Proxy
It runs on each node. It acts as a network proxy connecting locally running pods to outside world. It also functions as a load balancer (i.e. Services which provides a VIP and acts as load balancer) for groups of pods sharing the same label (i.e. if a node has multiple pods balances the traffic between those pods).

Just to understand kube-proxy, Here’s how Kubernetes services work! A service is a collection of pods, which each have their own IP address (like 10.1.0.3, 10.2.3.5, 10.3.5.6)

1. Every Kubernetes service gets an IP address (like 10.23.1.2)
2. kube-dns resolves Kubernetes service DNS names to IP addresses (so my-svc.my-namespace.svc.cluster.local might map to 10.23.1.2)
3. kube-proxy sets up iptables rules in order to do random load balancing between them.

So when you make a request to my-svc.my-namespace.svc.cluster.local, it resolves to 10.23.1.2, and then iptables rules on your local host (generated by kube-proxy) redirect it to one of 10.1.0.3 or 10.2.3.5 or 10.3.5.6 at random.

## DNS
As stated above, Pods are ephemeral, and because of this, their IP addresses are not reliable endpoints for communication. Although Services solve this by providing a stable address in front of a group of Pods,  consumers of the Service still want to avoid using an IP address. Kubernetes solves this by using DNS for service discovery.

The default internal domain name for a cluster is **cluster.local**. When you create a Service, it assembles a subdomain of **namespace.svc.cluster.local** (where namespace is the namespace in which the service is running) and sets its name as the hostname. For example, if the service was named *service-a* and ran in the *backend* namespace, consumers of the service would be able to reach it as *service-a.backend.svc.cluster.local*. If the service's IP changes, the hostname remains the same. There is no interruption of service.

![Alt text](/images/k8s-dns.jpg)

The default DNS provider for Kubernetes is KubeDNS, but it’s a pluggable component. Beginning with Kubernetes 1.11 CoreDNS is available as an alternative. In addition to providing the same basic DNS functionality within the cluster, CoreDNS supports a wide range of plugins to activate additional functionality.

## Route traffic to the cluster - Ingress Controller
Ingress Controller - An [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) is a Kubernetes resource that routes traffic from outside your cluster to services within the cluster. You must have an ingress controller like [F5](https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/v1.11/) or [HAProxy](https://github.com/haproxytech/kubernetes-ingress) to satisfy an Ingress. Only creating an Ingress resource has no effect.

### Best explained in this video
[**Kubernetes Ingress**](https://www.youtube.com/watch?v=VicH6KojwCI&feature=youtu.be)

## Kubernetes networking - joining the pieces together

![Alt text](/images/k8s-networking.jpg)

> cbr0 is Linux bridge that creates a veth pair for each pod with the host end of each pair connected to cbr0. Refer [kubenet network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#kubenet)

In the below example, we are creating AKS cluster in Azure with the advanced networking options
```bash
az aks create \
    --resource-group <cluster-resource-group> \
    --name <cluster-name> \
    --load-balancer-sku standard \
    --network-plugin azure \  # multi-host networking which allows pods to communicate with each other, accepted values: azure, kubenet
    --vnet-subnet-id <subnet-id> \  # subnet in an existing VNet into which to deploy the cluster
    --docker-bridge-address 172.17.0.1/16 \ # IP address and netmask for the Docker bridge
    --dns-service-ip 10.2.0.10 \    # IP address assigned to the Kubernetes DNS service 
    --service-cidr 10.2.0.0/24  # A CIDR notation IP range from which to assign service cluster IPs.
```
