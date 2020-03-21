# Definitions

### Namespaces
They provide process isolation. 
A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. Refer the below link for details  
http://man7.org/linux/man-pages/man7/namespaces.7.html

### Cgroups
They provide resource management. 
Control groups, usually referred to as cgroups, are a Linux kernel feature which allow how much resource can be utilized by the processes. Resources consumed by a process is limited and monitored through cgroups.   
http://man7.org/linux/man-pages/man7/cgroups.7.html

### Netfilter
Netfilter manages the rules that define network communication for the Linux kernel. These rules permit, deny, route, modify, and forward packets. It organizes these rules into tables according to their purpose. Netfilter achieves these rules by the command used to configure it: iptables.

### NAT
These rules control network address translation. They modify the source or destination address for the packet,
changing how the kernel routes the packet.

