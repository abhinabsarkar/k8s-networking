# Service
In Kubernetes pods are ephemeral. Kubernetes deploys applications through Pods. Pods can be placed on different hosts (nodes), scaled up by increasing their number, scaled down by killing the excess ones, and moved from one node to another. For example, the number of Pods in a ReplicaSet might change as the Deployment scales it up or down to accommodate changes in load on the application. All these dynamic actions must occur while ensuring that the application remains reachable at all times. To solve this problem, Services logically group pods to allow for direct access via an IP address or DNS name and on a specific port. The set of Pods targeted by a Service is usually determined by a selector. Selector allows users to filter a list of resources based on labels.

A sample Service kubernetes object can be seen below.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hellopython-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 5000
  selector:
    app: hellopython-app
  type: ClusterIP
```
In the above example, any client that wants to talk to hellopython-app, should send request to the service hellopython-svc or its FQDN **hellopython-svc.default.svc.cluster.local**

## Service Discovery
Services are assigned internal DNS entries by the Kubernetes DNS service. That means a client can call a backend service using the DNS name. The same mechanism can be used for service-to-service communication. The DNS entries are organized by namespace, so if your namespaces correspond to bounded contexts, then the DNS name for a service will map naturally to the application domain.

The following diagram show the conceptual relation between services and pods. The actual mapping to endpoint IP addresses and ports is done by kube-proxy, the Kubernetes network proxy.

![Alt Text](/images/k8s-service.jpg)

# Service features
* They can expose more than one port
* They support internal session affinity
* Services make use of health and readiness probes offered by Kubernetes to ensure that not only the Pods are in the running state, but also they return the expected response
* They allow specifying the IP address that that Service exposes by altering the spec.ClusterIP parameter

## Service Types (Publishing Services)
In Kubernetes, the Service component is used to provide a static URL through which a client can consume a service. The Service component is Kubernetes's way of handling more than one connectivity scenario.

**ClusterIP** - Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default ServiceType. It enables cluster resources to reach one another via a known address while maintaining the security boundaries of the cluster itself.

For example, a database used by a backend application does not need to be visible outside of the cluster, so using a service of type ClusterIP is appropriate. The backend application
would expose an API for interacting with records in the database, and a frontend application or remote clients would consume that API.

![Alt Text](/images/aks-clusterip.jpg)

**NodePort** - Exposes the Service on each Node’s IP at a static port (the NodePort). The range of available ports is a cluster-level configuration item, and the Service can either choose one of the ports at random
or have one designated in its configuration. A ClusterIP Service, to which the NodePort Service routes, is automatically created. You’ll be able to contact the NodePort Service, from outside the cluster, by requesting *NodeIP:NodePort*.

External load balancers frequently use NodePort services. They receive traffic for a specific site or address and forward it to the cluster on that specific port.

![Alt Text](/images/aks-nodeport.jpg)
> The diagram shows AKS node but the concept is native to kubernetes

**LoadBalancer** - Exposes the Service externally using a cloud provider’s load balancer. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created. It proxies the request to the corresponding Pods via NodePort and ClusterIP Services.

![Alt Text](/images/aks-loadbalancer.jpg)

Comparing the three Service types side-by-side together

![Alt Text](/images/service-type.jpg)

**ExternalName** - Creates a specific DNS entry for easier application access.
