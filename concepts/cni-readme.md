# Container Network Interface
The Container Networking Interface (CNI) provides a specification and a series of libraries for writing plugins to configure network interfaces in Linux containers. The specification requires that providers implement their plugin as a binary executable that the container engine invokes.

The CNI specification expects the container runtime to create a new network namespace before invoking the CNI plugin. The plugin is then responsible for connecting the container’s network with that of the host. It does this by creating virtual Ethernet devices (veth).

## Kubernetes & CNI

Kubernets adopts the Container Network Interface (CNI) model to provide a contract between networks and containers. Kubernetes does this via the Kubelet process running on each node of the cluster.

To use the CNI plugin, pass **--network-plugin=cni** to the Kubelet when launching it. If your environment is not using the default configuration directory (/etc/cni/net.d), pass the correct configuration directory as a value to --cni-conf-dir. The Kubelet looks for the CNI plugin binary at /opt/cni/bin, but you can specify an alternative location with --cni-bin-dir.

The CNI plugin provides IP address management (IPAM) for the Pods and builds routes for the virtual interfaces. To do this, the plugin interfaces with an IPAM plugin that is also part of the CNI specification. The IPAM plugin must also be a single executable that the CNI plugin consumes. The role of the IPAM plugin is to provide to the CNI plugin the gateway, IP subnet, and routes for the Pod.

From a user perspective, provisioning networking for a container involves two steps:
* Define the network JSON
* Connect container to the network

Internally, CNI provisioning involves three steps:
* Runtime creates a network namespace and gives it a name
* Invokes the CNI plugin specified in the “type” field of the network JSON. Type field refers to the plugin being used and so CNI invokes the corresponding binary
* Plugin code in turn will create a veth pair, check the IPAM type and data in the JSON

You must deploy a Container Network Interface (CNI) based Pod network add-on so that your Pods can communicate with each other. You can read more about CNI in the below link  
https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#cni


Some of the popular Pod network add ons are listed below:
* Calico
* Flannel
* Waeve Net

Refer the below link for installing add ons  
https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy