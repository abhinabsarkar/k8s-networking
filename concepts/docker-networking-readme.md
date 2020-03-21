# Docker networking

## Docker networking types
When a Docker container launches, the Docker engine assigns it a network interface with an IP address, a default gateway, and other components, such as a routing table and DNS services. Bridge is the default network that containers connect to if no other network is specified (the analog on Windows is the nat network type).

### Bridge network 
The container runs in a private network internal to the host. Communication is open to other containers in the same network. Communication with services outside of the host goes through network address translation (NAT) before exiting the host.

![Alt text](/images/bridge-network.jpg)

### Lets get a closer look to the bridge network
In a standard Docker installation, the Docker daemon creates a bridge on the host with the name of **docker0** (in the above diagram docker0 is assigned an IP 172.17.42.1/16). When a container launches, Docker then creates a virtual ethernet device for it. This device appears within the container (in the above diagram, containers with ip 172.17.0.1/16 & 172.17.0.2/16) as **eth0** and on the host with a name like **vethxxx** where **xxx** is a unique identifier for the interface (in the above diagram veth27e6b05 & veth0f00eed). The vethxxx interface is added to the docker0 bridge, and this enables communication with other containers on the same host that also use the default bridge. The docker host in the above diagram is assigned an ip 192.168.178.100 & it is connected to the switch having ip as 192.168.178.0/24.

#### Demo
>The below demo is done on windows 10 OS Version 10.0.17134 N/A Build 17134, using docker-desktop v19.03.8 running linux containers.

Run the following command on a host with Docker installed. Since we are not specifying the network switch - the container will connect to the default bridge when it launches.
```bash
# run a busybox container instance
docker run -it --rm busybox /bin/sh
```
Run the below commands & it will show the IP address of the container "172.17.0.3/16" with the eth0 interface.
```bash
# run the ip address command
/ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop qlen 1000
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
# run the ip route command
/ ip route
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link  src 172.17.0.3
```
To check the network interface (like docker0, ip address) on the host, we need to login to the linux VM. Docker-desktop for windows run on a VM called as MobyLinuxVM, but you cannot login to that VM via Hyper-V Manager. Since you can't SSH into the VM, we’ll create a container that has full root access and then access the file system from there.
```bash
# Get container with access to Docker Daemon
docker run --privileged -it -v /var/run/docker.sock:/var/run/docker.sock jongallant/ubuntu-docker-client
# Run container with full root access
docker run --net=host --ipc=host --uts=host --pid=host -it --security-opt=seccomp=unconfined --privileged --rm -v /:/host alpine /bin/sh
# Switch to host file system
chroot /host
# Check the hostname of the linux VM
/ hostname
docker-desktop
```

Now connected to the host, run the below command
```bash
/ ip addr
```

You can see the docker0 bridge in the network interface, along with its IP.

![Alt text](/images/docker0.jpg)

To check the corresponding veth interface in the host for the container's veth0 interface, it can be found out by matching a container interface’s iflink value with a host veth interface’s ifindex value.

First run the bridge control command on the docker host.
```bash
# run brctl on the host machine
/ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024210e3b1d9       no              vethcf88ac1
                                                        veth3a6d989
cni0            8000.561fa52fe2fb       no              vethab1c5aff
                                                        vethe9888e1c
```                            
On the container, run :
```bash
cat /sys/class/net/eth0/iflink
16
```
And on the host, find a veth with an ifindex value matching the iflink value of your container’s interface :
```bash
cat /sys/class/net/vethXXXXXXX/ifindex
# Eg:
cat /sys/class/net/vethcf88ac1/ifindex
7
cat /sys/class/net/veth3a6d989/ifindex
16
```
In this case, veth3a6d989 on the host linux machine corresponds to the veth0 on the container.

Although Docker mapped the container IPs on the bridge, network services running inside of the container are not visible outside of the host. To make them visible, the Docker Engine must be told when launching a container to map ports from that container to ports on the host. This process is called publishing. For example, if you want to map port 80 of a container to port 8080 on the host, then you would have to publish the port as shown in the following command:
```bash
docker run --name nginx -p 8080:80 nginx
```
By default, the Docker container can send traffic to any destination. The Docker daemon creates a rule within **Netfilter (Linux kernel firewall - known more commonly by the command used to configure it: iptables.)** that modifies outbound packets and changes the source address to be the address of the host itself. The Netfilter configuration allows inbound traffic via the rules that Docker creates when initially publishing the container's ports.

### Custom Bridge Network

https://docs.docker.com/engine/tutorials/networkingcontainers/#create-your-own-bridge-network

### Overlay Driver Network Architecture

Refer this article for details - https://success.docker.com/article/networking

### Packet flow in an overlay network
![Alt text](/images/packetwalk.jpg)

## Docker Networking – key points
Docker adopts the Container Network Model (CNM), providing the following contract between networks and containers:
* All containers on the same network can communicate freely with each other
* Multiple networks are the way to segment traffic between containers and should be supported by all drivers
* Multiple endpoints per container are the way to join a container to multiple networks
* An endpoint is added to a network sandbox to provide it with network connectivity
* Docker Engine can create overlay networks on a single host. Docker Swarm can create overlay networks that span hosts in the cluster
* A container can be assigned an IP on an overlay network. Containers that use the same overlay network can communicate, even if they are running on different hosts

By default, nodes in the swarm encrypt traffic between themselves and other nodes.
Connections between nodes are automatically secured through TLS authentication with
certificates