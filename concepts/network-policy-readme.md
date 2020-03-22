# Network policy
In an enterprise deployment of Kubernetes the cluster often supports multiple projects with different goals. Each of these projects has different workloads, and each of these might require a different security policy.

Pods, by default, do not filter incoming traffic. There are no firewall rules for inter-Pod communication. Instead, this responsibility falls to the **NetworkPolicy** resource, which uses a specification to define the network rules applied to a set of Pods. For eg: **if a cluster is not using a network policy, any Pod can talk to any other Pod**. Nothing prevents a web Pod from communicating directly with the database Pods. 

![Alt text](/images/no-network-policy.jpg)

If the security requirements of the cluster dictate a need for clear separation between tiers, a network policy enforces it.

For eg: the policy defined below states that the database Pods can only receive traffic from the Pods with the labels *app=myapp*  and *role=backend*. It also defines that the backend Pods can only receive traffic from Pods with the labels *app=myapp* and *role=web*.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-access-ingress
spec:
  podSelector:
    matchLabels:
      app: myapp
      role: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
          role: web
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: db-access-ingress
spec:
  podSelector:
  matchLabels:
    app: myapp
    role: db
  ingress:
  - from:
    - podSelector:
      matchLabels:
        app: myapp
        role: backend
```
With this network policy in place, Kubernetes blocks communication between the web and database tiers.

![Alt text](/images/network-policy.jpg)

## How a Network Policy Works
In addition to the fields used by all Kubernetes manifests, the specification of the NetworkPolicy resource requires some extra fields.

### podSelector
This field tells Kubernetes how to find the Pods to which this policy applies. Multiple network policies can select the same set of Pods, and the ingress rules are applied sequentially. The field is not optional, but if the manifest defines a key with no value, it applies to all Pods in the namespace.

### policyTypes
This field defines the direction of network traffic to which the rules apply. If missing, Kubernetes  interprets the rules and only applies them to ingress traffic unless egress rules also appear in the rules list. This default interpretation simplifies the manifest's definition by having it adapt to the rules defined later.  
Because Kubernetes always defines an ingress policy if this field is unset, a network policy for egress-only rules must explicitly define the policyType of Egress.