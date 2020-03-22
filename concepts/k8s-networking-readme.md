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
Pods are ephemeral. Since Kubernetes can terminate Pods at any time, they are unreliable endpoints for direct communication. For example, the number of Pods in a ReplicaSet might change as the Deployment scales it up or down to accommodate changes in load on the application. Hence, Kubernetes offers the Service resource, which provides a stable IP address and balances traffic across all of the Pods behind it. Services which sit in front of Pods use a selector and labels to find the Pods they manage. All Pods with a label that matches the selector receive traffic through the Service. Like a traditional load balancer, the service can expose the Pod functionality at any port, irrespective of the port in use by the Pods themselves.

![Alt text](/images/k8s-service.jpg)

## Kubernetes Service Types
[Service Types](https://github.com/abhinabsarkar/aks/blob/master/concepts/service-readme.md#service-types-publishing-services)

## Kubelet
It is the primary “node agent” that runs on each node. They ensure the containers described in the PodSpecs (YAML file describing the Pod) are running and healthy.

## Kube Proxy
It runs on each node. It acts as a network proxy connecting locally running pods to outside world. It also functions as a load balancer (i.e. Services which provides a VIP and acts as load balancer) for groups of pods sharing the same label (i.e. if a node has multiple pods balances the traffic between those pods).

## DNS
As stated above, Pods are ephemeral, and because of this, their IP addresses are not reliable endpoints for communication. Although Services solve this by providing a stable address in front of a group of Pods,  consumers of the Service still want to avoid using an IP address. Kubernetes solves this by using DNS for service discovery.

The default internal domain name for a cluster is **cluster.local**. When you create a Service, it assembles a subdomain of **namespace.svc.cluster.local** (where namespace is the namespace in which the service is running) and sets its name as the hostname. For example, if the service was named *service-a* and ran in the *backend* namespace, consumers of the service would be able to reach it as *service-a.backend.svc.cluster.local*. If the service's IP changes, the hostname remains the same. There is no interruption of service.

![Alt text](/images/k8s-dns.jpg)

The default DNS provider for Kubernetes is KubeDNS, but it’s a pluggable component. Beginning with Kubernetes 1.11 CoreDNS is available as an alternative. In addition to providing the same basic DNS functionality within the cluster, CoreDNS supports a wide range of plugins to activate additional functionality.
