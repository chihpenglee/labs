## <a name="challenges"></a>Challenges of Networking Containers and Microservices

Microservices practices have increased the scale of applications which has put even more importance on the methods of connectivity and isolation that we provide to applications. The Docker networking philosophy is application driven. It aims to provide options and flexibility to the network operators as well as the right level of abstraction to the application developers. 

Like any design, network design is a balancing act. __Docker Datacenter__ and the Docker ecosystem provides multiple tools to network engineers to achieve the best balance for their applications and environments. Each option provides different benefits and tradeoffs. The remainder of this guide details each of these choices so network engineers can understand what might be best for their environments.

Docker has developed a new way of delivering applications, and with that, containers have also changed some aspects of how we approach networking. The following topics are common design themes for containerized applications:

- __Portability__
	- _How do I guarantee maximum portability across diverse network environments while taking advantage of unique network characteristics?_

- __Service Discovery__ 
	- _How do I know where services are living as they are scaled up and down?_

- __Load Balancing__
	- _How do I share load across services as services themselves are brought up and scaled?_

- __Security__ 
	- _How do I segment to prevent the right containers from accessing each other?_
	- _How do I guarantee that a container with application and cluster control traffic is secure?_

- __Performance__  
 	- _How do I provide advanced network services while minimizing latency and maximizing bandwidth?_

- __Scalability__  
 	- _How do I ensure that none of these characteristics are sacrificed when scaling applications across many hosts?_

### <a name="concepts"></a>Concepts
This section contains 14 different short networking concept chapters. Feel free to skip right to the [tutorials](../tutorials.md) if you feel you are ready and come back here if you need a refresher. The concept chapters are:

1. [The Container Networking Model](01-cnm.md)

1. [Drivers](02-drivers.md)

1. [Linux Networking Fundamentals](03-linux-networking.md)

1. [Docker Network Control Plane](04-docker-network-cp.md)

1. [Bridge Networks](05-bridge-networks.md)

1. [Overlay Networks](06-overlay-networks.md)

1. [MACVLAN](07-macvlan.md)

1. [Host (Native) Network Driver](08-host-networking.md)

1. [Physical Network Design Requirements](09-physical-networking.md)

1. [Load Balancing Design Considerations](10-load-balancing.md)

1. [Security](11-security.md)

1. [IP Address Management](12-ipaddress-management.md)

1. [Troubleshooting](13-troubleshooting.md)

1. [Network Deployment Models](14-network-models.md)

## <a name="cnm"></a>The Container Networking Model
The Docker networking architecture is built on a set of interfaces called the _Container Networking Model_ (CNM). The philosophy of CNM is to provide application portability across diverse infrastructures. This model strikes a balance to achieve application portability and also takes advantage of special features and capabilities of the infrastructure. 

![Container Networking Model](./img/cnm.png)

### CNM Constructs
There are several high-level constructs in the CNM. They are all OS and infrastructure agnostic so that applications can have a uniform experience no matter the infrastructure stack.

 - __Sandbox__ — A Sandbox contains the configuration of a container's network stack. This includes management of the container's interfaces, routing table, and DNS settings. An implementation of a Sandbox could be a Linux Network Namespace, a FreeBSD Jail, or other similar concept. A Sandbox may contain many endpoints from multiple networks.
 - __Endpoint__ — An Endpoint joins a Sandbox to a Network. The Endpoint construct exists so the actual connection to the network can be abstracted away from the application. This helps maintain portability so that a service can use different types of network drivers without being concerned with how it's connected to that network.
 - __Network__ — The CNM does not specify a Network in terms of the OSI model. An implementation of a Network could be a Linux bridge, a VLAN, etc. A Network is a collection of endpoints that have connectivity between them. Endpoints that are not connected to a network will not have connectivity on a Network.

Next: **[Drivers](02-drivers.md)**
## CNM Driver Interfaces
The Container Networking Model provides two pluggable and open interfaces that can be used by users, the community, and vendors to leverage additional functionality, visibility, or control in the network.

### Categories of Network Drivers

 - __Network Drivers__ — Docker Network Drivers provide the actual implementation that makes networks work. They are pluggable so that different drivers can be used and interchanged easily to support different use-cases. Multiple network drivers can be used on a given Docker Engine or Cluster concurrently, but each Docker network is only instantiated through a single network driver. There are two broad types of CNM network drivers:
 	- __Built-In Network Drivers__ — Built-In Network Drivers are a native part of the Docker Engine and are provided by Docker. There are multiple to choose from that support different capabilities like overlay networks or local bridges.
 	- __Plug-In Network Drivers__ — Plug-In Network Drivers are network drivers created by the community and other vendors. These drivers can be used to provide integration with incumbent software and hardware. Users can also create their own drivers in cases where they desire specific functionality that is not supported by an existing network driver.
 - __IPAM Drivers__ — Docker has a built-in IP Address Management Driver that provides default subnets or IP addresses for Networks and Endpoints if they are not specified. IP addressing can also be manually assigned through network, container, and service create commands. Plug-In IPAM drivers also exist that provide integration to existing IPAM tools. 

### Docker Built-In Network Drivers
The Docker built-in network drivers are part of Docker Engine and don't require any extra modules. They are invoked and used through standard `docker network` commands. The follow built-in network drivers exist:

- __Bridge__ — The `bridge` driver creates a Linux bridge on the host that is managed by Docker. By default containers on a bridge will be able to communicate with each other. External access to containers can also be configured through the `bridge` driver. 

- __Overlay__ — The `overlay` driver creates an overlay network that supports multi-host networks out of the box. It uses a combination of local Linux bridges and VXLAN to overlay container-to-container communications over physical network infrastructure. 

- __MACVLAN__ — The `macvlan` driver uses the MACVLAN bridge mode to establish a connection between container interfaces and a parent host interface (or sub-interfaces). It can be used to provide IP addresses to containers that are routable on the physical network. Additionally VLANs can be trunked to the `macvlan` driver to enforce Layer 2 container segmentation.

- __Host__ — With the `host` driver, a container uses the networking stack of the host. There is no namespace separation, and all interfaces on the host can be used directly by the container.

- __None__ — The `none` driver gives a container its own networking stack and network namespace but does not configure interfaces inside the container. Without additional configuration, the container is completely isolated from the host networking stack.


### Default Docker Networks
By default a `none`, `host`, and `bridge` network will exist on every Docker host. These networks cannot be removed. When instantiating a Swarm, two additional networks, a bridge network named `docker_gwbridge` and an overlay network named `ingress`, are automatically created to facilitate cluster networking. 

The `docker network ls` command shows these default Docker networks for a Docker Swarm:

```
NETWORK ID          NAME                DRIVER              SCOPE
1475f03fbecb        bridge              bridge              local
e2d8a4bd86cb        docker_gwbridge     bridge              local
407c477060e7        host                host                local
f4zr3zrswlyg        ingress             overlay             swarm
c97909a4b198        none                null                local
```

In addition to these default networks, [user defined networks](#userdefined) can also be created. They are discussed later in this document.

### Network Scope
As seen in the `docker network ls` output, Docker network drivers have a concept of _scope_. The network scope is the domain of the driver which can be the `local` or `swarm` scope. Local scope drivers provide connectivity and network services (such as DNS or IPAM) within the scope of the host. Swarm scope drivers provide connectivity and network services across a swarm cluster. Swarm scope networks will have the same network ID across the entire cluster while local scope networks will have a unique network ID on each host. 

### Docker Plug-In Network Drivers
The following community- and vendor-created plug-in network drivers are compatible with CNM. Each provides unique capabilities and network services for containers.

| Driver | Description   |
|------|------|
| [**contiv**](http://contiv.github.io/) | An open source network plugin led by Cisco Systems to provide infrastructure and security policies for multi-tenant microservices deployments. Contiv also provides integration for non-container workloads and with physical networks, such as ACI. Contiv implements plug-in network and IPAM drivers. |
| [**weave**](https://www.weave.works/docs/net/latest/introducing-weave/) |  A network plugin that creates a virtual network that connects Docker containers across multiple hosts or clouds. Weave provides automatic discovery of applications, can operate on partially connected networks, does not require an external cluster store, and is operations friendly.   |
| [**calico**](https://www.projectcalico.org/)     | Calico is an open source solution for virtual networking in cloud datacenters.  It targets datacenters where most of the workloads (VMs, containers, or bare metal servers) only require IP connectivity. Calico provides this connectivity using standard IP routing. Isolation between workloads — whether according to tenant ownership, or any finer grained policy — is achieved via iptables programming on the servers hosting the source and destination workloads.  |
| [**kuryr**](https://github.com/openstack/kuryr)    | A network plugin developed as part of the OpenStack Kuryr project. It implements the Docker networking (libnetwork) remote driver API by utilizing Neutron, the OpenStack networking service. Kuryr includes an IPAM driver as well. |

### Docker Plug-In IPAM Drivers
Community and vendor created IPAM drivers can also be used to provide integrations with existing systems or special capabilities.

| Driver | Description   |
|------|------|
| [**infoblox**](https://store.docker.com/community/images/infoblox/ipam-driver) | An open source IPAM plugin that provides integration with existing Infoblox tools. |

> There are many Docker plugins that exist and more are being created all the time. Docker maintains a list of the [most common plugins.](https://docs.docker.com/engine/extend/legacy_plugins/)

Next: **[Linux Network Fundamentals](03-linux-networking.md)**
## <a name="drivers"></a><a name="linuxnetworking"></a>Linux Network Fundamentals

The Linux kernel features an extremely mature and performant implementation of the TCP/IP stack (in addition to other native kernel features like DNS and VXLAN). Docker networking uses the kernel's networking stack as low level primitives to create higher level network drivers. Simply put, _Docker networking <b>is</b> Linux networking._ 

This implementation of existing Linux kernel features ensures high performance and robustness. Most importantly, it provides portability across many distributions and versions which enhances application portability.

There are several Linux networking building blocks which Docker uses to implement its built-in CNM network drivers. This list includes **Linux bridges**, **network namespaces**, **veth pairs**,  and **iptables**. The combination of these tools implemented as network drivers provide the forwarding rules, network segmentation, and management tools for complex network policy.

### <a name="linuxbridge"></a>The Linux Bridge
A **Linux bridge** is a Layer 2 device that is the virtual implementation of a physical switch inside the Linux kernel. It forwards traffic based on MAC addresses which it learns dynamically by inspecting traffic. Linux bridges are used extensively in many of the Docker network drivers. A Linux bridge is not to be confused with the `bridge` Docker network driver which is a higher level implementation of the Linux bridge.


### Network Namespaces
A Linux **network namespace** is an isolated network stack in the kernel with its own interfaces, routes, and firewall rules. It is a security aspect of containers and Linux, used to isolate containers. In networking terminology they are akin to a VRF that segments the network control and data plane inside the host. Network namespaces ensure that two containers on the same host will not be able to communicate with each other or even the host itself unless configured to do so via Docker networks. Typically, CNM network drivers implement separate namespaces for each container. However, containers can share the same network namespace or even be a part of the host's network namespace. The host network namespace contains the host interfaces and host routing table. This network namespace is called the global network namespace.

### Virtual Ethernet Devices
A **virtual ethernet device** or **veth** is a Linux networking interface that acts as a connecting wire between two network namespaces. A veth is a full duplex link that has a single interface in each namespace. Traffic in one interface is directed out the other interface. Docker network drivers utilize veths to provide explicit connections between namespaces when Docker networks are created. When a container is attached to a Docker network, one end of the veth is placed inside the container (usually seen as the `ethX` interface) while the other is attached to the Docker network. 

### iptables
**`iptables`** is the native packet filtering system that has been a part of the Linux kernel since version 2.4. It's a feature rich L3/L4 firewall that provides rule chains for packet marking, masquerading, and dropping. The built-in Docker network drivers utilize `iptables` extensively to segment network traffic, provide host port mapping, and to mark traffic for load balancing decisions.

Next: **[Docker Network Control Plane](04-docker-network-cp.md)**
## <a name="controlplane"></a>Docker Network Control Plane
The Docker-distributed network control plane manages the state of Swarm-scoped Docker networks in addition to propagating control plane data. It is a built-in capability of Docker Swarm clusters and does not require any extra components such as an external KV store. The control plane uses a [Gossip](https://en.wikipedia.org/wiki/Gossip_protocol) protocol based on [SWIM](https://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf) to propagate network state information and topology across Docker container clusters. The Gossip protocol is highly efficient at reaching eventual consistency within the cluster while maintaining constant rates of message size, failure detection times, and convergence time across very large scale clusters. This ensures that the network is able to scale across many nodes without introducing scaling issues such as slow convergence or false positive node failures. 

The control plane is highly secure, providing confidentiality, integrity, and authentication through encrypted channels. It is also scoped per network which greatly reduces the updates that any given host will receive. 

![Docker Network Control Plane](./img/gossip.png)

It is composed of several components that work together to achieve fast convergence across large scale networks. The distributed nature of the control plane ensures that cluster controller failures don't affect network performance. 

The Docker network control plane components are as follows:

- **Message Dissemination** updates nodes in a peer-to-peer fashion fanning out the information in each exchange to a larger group of nodes. Fixed intervals and size of peer groups ensures that network usage is constant even as the size of the cluster scales. Exponential information propagation across peers ensures that convergence is fast and bounded across any cluster size.
- **Failure Detection** utilizes direct and indirect hello messages to rule out network congestion and specific paths from causing false positive node failures. 
- **Full State Syncs** occur periodically to achieve consistency faster and resolve network partitions.
- **Topology Aware** algorithms understand the relative latency between themselves and other peers. This is used to optimize the peer groups which makes convergence faster and more efficient. 
- **Control Plane Encryption** protects against man in the middle and other attacks that could compromise network security.

> The Docker Network Control Plane is a component of [Swarm](https://docs.docker.com/engine/swarm/) and requires a Swarm cluster to operate.

Next: **[Docker Bridge Network Driver Architecture](05-bridge-networks.md)**
## <a name="drivers"></a>Docker Bridge Network Driver Architecture

This section explains the default Docker bridge network as well as user-defined bridge networks.

### Default Docker Bridge Network
On any host running Docker Engine, there will, by default, be a local Docker network named `bridge`. This network is created using a `bridge` network driver which instantiates a Linux bridge called `docker0`. This may sound confusing. 

- `bridge` is the name of the Docker network
- `bridge` is the network driver, or template, from which this network is created
- `docker0` is the name of the Linux bridge that is the kernel building block used to implement this network

On a standalone Docker host, `bridge` is the default network that containers will connect to if no other network is specified. In the following example a container is created with no network parameters. Docker Engine connects it to the `bridge` network by default. Inside the container we can see `eth0` which is created by the `bridge` driver and given an address by the Docker built-in IPAM driver.


```bash
#Create a busybox container named "c1" and show its IP addresses
host$ docker run -it --name c1 busybox sh
c1 # ip address
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
...
```
> A container interface's MAC address is dynamically generated and embeds the IP address to avoid collision. Here `ac:11:00:02` corresponds to `172.17.0.2`.

By using the tool `brctl` on the host, we show the Linux bridges that exist in the host network namespace. It shows a single bridge called `docker0`. `docker0` has one interface, `vetha3788c4`, which provides connectivity from the bridge to the `eth0` interface inside container `c1`.

```
host$ brctl show
bridge name		 bridge id			  STP enabled    interfaces
docker0		     8000.0242504b5200	  no       		 vethb64e8b8
```

Inside container `c1` we can see the container routing table that directs traffic to `eth0` of the container and thus the `docker0` bridge.

```bash
c1# ip route
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0  src 172.17.0.2
```
A container can have zero to many interfaces depending on how many networks it is connected to. Each Docker network can only have a single interface per container.

![Default Docker Bridge Network](./img/bridge1.png)

When we peek into the host routing table we can see the IP interfaces in the global network namespace that now includes `docker0`. The host routing table provides connectivity between `docker0` and `eth0` on the external network, completing the path from inside the container to the external network.

```bash
host$ ip route
default via 172.31.16.1 dev eth0
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.42.1
172.31.16.0/20 dev eth0  proto kernel  scope link  src 172.31.16.102
```

By default `bridge` will be assigned one subnet from the ranges 172.[17-31].0.0/16 or 192.168.[0-240].0/20 which does not overlap with any existing host interface. The default `bridge` network can be also be configured to use user-supplied address ranges. Also, an existing Linux bridge can be used for the `bridge` network rather than Docker creating one. Go to the [Docker Engine docs](https://docs.docker.com/engine/userguide/networking/default_network/custom-docker0/) for more information about customizing `bridge`. 

>  The default `bridge` network is the only network that supports legacy [links](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/). Name-based service discovery and user-provided IP addresses are __not__ supported by the default `bridge` network.



### <a name="userdefined"></a>User-Defined Bridge Networks
In addition to the default networks, users can create their own networks called **user-defined networks** of any network driver type. In the case of user-defined `bridge` networks, Docker will create a new Linux bridge on the host. Unlike the default `bridge` network, user-defined networks supports manual IP address and subnet assignment. If an assignment isn't given, then Docker's default IPAM driver will assign the next subnet available in the private IP space. 

![User-Defined Bridge Network](./img/bridge2.png)

Below we are creating a user-defined `bridge` network and attaching two containers to it. We specify a subnet and call the network `my_bridge`. One container is not given IP parameters, so the IPAM driver assigns it the next available IP in the subnet. The other container has its IP specified.

```
$ docker network create -d bridge my_bridge
$ docker run -itd --name c2 --net my_bridge busybox sh
$ docker run -itd --name c3 --net my_bridge --ip 10.0.0.254 busybox sh
```

`brctl` now shows a second Linux bridge on the host. The name of the Linux bridge, `br-4bcc22f5e5b9`, matches the Network ID of the `my_bridge` network. `my_bridge` also has two `veth` interfaces connected to containers `c2` and `c3`. 

```
$ brctl show
bridge name		 bridge id			  STP enabled    interfaces
br-b5db4578d8c9	 8000.02428d936bb1	  no		     vethc9b3282
							                         vethf3ba8b5
docker0		     8000.0242504b5200	  no		     vethb64e8b8

$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
b5db4578d8c9        my_bridge           bridge              local
e1cac9da3116        bridge              bridge              local
...
```

Listing the global network namespace interfaces shows the Linux networking circuitry that's been instantiated by Docker Engine. Each `veth` and Linux bridge interface appears as a link between one of the Linux bridges and the container network namespaces.

```bash
$ ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 
5: vethb64e8b8@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 
6: br-b5db4578d8c9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 
8: vethc9b3282@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 
10: vethf3ba8b5@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 
...
```

### External and Internal Connectivity
By default all containers on the same `bridge` driver network will have connectivity with each other without extra configuration. This is an aspect of most types of Docker networks. By virtue of the Docker network the containers are able to communicate across their network namespaces and (for multi-host drivers) across external networks as well. **Communication between different Docker networks is firewalled by default.** This is a fundamental security aspect that allows us to provide network policy using Docker networks. For example, in the figure above containers `c2` and `c3` have reachability but they cannot reach `c1`.

Docker `bridge` networks are not exposed on the external (underlay) host network by default. Container interfaces are given IPs on the private subnets of the bridge network. Containers communicating with the external network are port mapped or masqueraded so that their traffic uses an IP address of the host. The example below shows outbound and inbound container traffic passing between the host interface and a user-defined `bridge` network.

![Port Mapping and Masquerading](./img/nat.png)

Outbound (egress) container traffic is allowed by default. Egress connections initiated by containers are masqueraded/SNATed to an ephemeral port (_typically in the range of 32768 to 60999_). Return traffic on this connection is allowed, and thus the container uses the best routable IP address of the host on the ephemeral port.

Ingress container access is provided by explicitly exposing ports. This port mapping is done by Docker Engine and can be controlled through UCP or the Engine CLI. A specific or randomly chosen port can be configured to expose a service or container. The port can be set to listen on a specific (or all) host interfaces, and all traffic will be mapped from this port to a port and interface inside the container.

This previous diagram shows how port mapping and masquerading takes place on a host. Container `C2` is connected to the `my_bridge` network and has an IP address of `10.0.0.2`. When it initiates outbound traffic the traffic will be masqueraded so that it is sourced from ephemeral port `32768` on the host interface `192.168.0.2`. Return traffic will use the same IP address and port for its destination and will be masqueraded internally back to the container address:port `10.0.0.2:33920`. 

Exposed ports can be configured using `--publish` in the Docker CLI or UCP. The diagram shows an exposed port with the container port `80` mapped to the host interface on port `5000`. The exposed container would be advertised at `192.168.0.2:5000`, and all traffic going to this interface:port would be sent to the container at `10.0.0.2:80`.


Next: **[Overlay Driver Network Architecture](06-overlay-networks.md)**

## <a name="overlaydriver"></a>Overlay Driver Network Architecture

The built-in Docker `overlay` network driver radically simplifies many of the challenges in multi-host networking. With the `overlay` driver, multi-host networks are first-class citizens inside Docker without external provisioning or components. `overlay` uses the Swarm-distributed control plane to provide centralized management, stability, and security across very large scale clusters.

### VXLAN Data Plane
The `overlay` driver utilizes an industry-standard VXLAN data plane that decouples the container network from the underlying physical network (the _underlay_). The Docker overlay network encapsulates container traffic in a VXLAN header which allows the traffic to traverse the physical Layer 2 or Layer 3 network. The overlay makes network segmentation dynamic and easy to control no matter what the underlying physical topology. Use of the standard IETF VXLAN header promotes standard tooling to inspect and analyze network traffic.

> VXLAN has been a part of the Linux kernel since version 3.7, and Docker uses the native VXLAN features of the kernel to create overlay networks. The Docker overlay datapath is entirely in kernel space. This results in fewer context switches, less CPU overhead, and a low-latency, direct traffic path between applications and the physical NIC. 

IETF VXLAN ([RFC 7348](https://datatracker.ietf.org/doc/rfc7348/)) is a data-layer encapsulation format that overlays Layer 2 segments over Layer 3 networks. VXLAN is designed to be used in standard IP networks and can support large-scale, multi-tenant designs on shared physical network infrastructure. Existing on-premises and cloud-based networks can support VXLAN transparently. 

VXLAN is defined as a MAC-in-UDP encapsulation that places container Layer 2 frames inside an underlay IP/UDP header. The underlay IP/UDP header provides the transport between hosts on the underlay network. The overlay is the stateless VXLAN tunnel that exists as point-to-multipoint connections between each host participating in a given overlay network. Because the overlay is independent of the underlay topology, applications become more portable. Thus, network policy and connectivity can be transported with the application whether it is on-premises, on a developer desktop, or in a public cloud.

![Packet Flow for an Overlay Network](./img/packetwalk.png)

In this diagram we see the packet flow on an overlay network. Here are the steps that take place when `c1` sends `c2` packets across their shared overlay network:

- `c1` does a DNS lookup for `c2`. Since both containers are on the same overlay network the Docker Engine local DNS server resolves `c2` to its overlay IP address `10.0.0.3`.
- An overlay network is a L2 segment so `c1` generates an L2 frame destined for the MAC address of `c2`.
- The frame is encapsulated with a VXLAN header by the `overlay` network driver. The distributed overlay control plane manages the locations and state of each VXLAN tunnel endpoint so it knows that `c2` resides on `host-B` at the physical address of `192.168.1.3`. That address becomes the destination address of the underlay IP header.
- Once encapsulated the packet is sent. The physical network is responsible of routing or bridging the VXLAN packet to the correct host.
- The packet arrives at the `eth0` interface of `host-B` and is decapsulated by the `overlay` network driver. The original L2 frame from `c1` is passed to the `c2`'s `eth0` interface and up to the listening application.



### Overlay Driver Internal Architecture
The Docker Swarm control plane automates all of the provisioning for an overlay network. No VXLAN configuration or Linux networking configuration is required. Data-plane encryption, an optional feature of overlays, is also automatically configured by the overlay driver as networks are created. The user or network operator only has to define the network (`docker network create -d overlay ...`) and attach containers to that network.
 
![Overlay Network Created by Docker Swarm](./img/overlayarch.png)

During overlay network creation, Docker Engine creates the network infrastructure required for overlays on each host. A Linux bridge is created per overlay along with its associated VXLAN interfaces. The Docker Engine intelligently instantiates overlay networks on hosts only when a container attached to that network is scheduled on the host. This prevents sprawl of overlay networks where connected containers do not exist.

In the following example we create an overlay network and attach a container to that network. We'll then see that Docker Swarm/UCP automatically creates the overlay network.

```bash
#Create an overlay named "ovnet" with the overlay driver
$ docker network create -d overlay ovnet

#Create a service from an nginx image and connect it to the "ovnet" overlay network
$ docker service create --network ovnet --name container nginx
```

When the overlay network is created, you will notice that several interfaces and bridges are created inside the host.

```bash
# Run the "ifconfig" command inside the nginx container
$ docker exec -it container ifconfig

#docker_gwbridge network
eth1      Link encap:Ethernet  HWaddr 02:42:AC:12:00:04
          inet addr:172.18.0.4  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe12:4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

#overlay network
eth2      Link encap:Ethernet  HWaddr 02:42:0A:00:00:07
          inet addr:10.0.0.7  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::42:aff:fe00:7/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)
     
#container loopback
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:48 errors:0 dropped:0 overruns:0 frame:0
          TX packets:48 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:4032 (3.9 KiB)  TX bytes:4032 (3.9 KiB)
```

Two interfaces have been created inside the container that correspond to two bridges that now exist on the host. On overlay networks, each container will have at least two interfaces that connect it to the `overlay` and the `docker_gwbridge`.

| Bridge | Purpose  |
|:------:|------|
| **overlay** | The ingress and egress point to the overlay network that VXLAN encapsulates and (optionally) encrypts traffic going between containers on the same overlay network. It extends the overlay across all hosts participating in this particular overlay. One will exist per overlay subnet on a host, and it will have the same name that a particular overlay network is given.    |
| **docker_gwbridge** |   The egress bridge for traffic leaving the cluster. Only one `docker_gwbridge` will exist per host. Container-to-Container traffic is blocked on this bridge allowing ingress/egress traffic flows only.      |



> The Docker Overlay driver has existed since Docker Engine 1.9, and an external K/V store was required to manage state for the network. Docker 1.12 integrated the control plane state into Docker Engine so that an external store is no longer required. 1.12 also introduced several new features including encryption and service load balancing. Networking features that are introduced require a Docker Engine version that supports them, and using these features with older versions of Docker Engine is not supported.

Next: **[MACVLAN](07-macvlan.md)**
## <a name="macvlandriver"></a>MACVLAN 

The `macvlan` driver is a new implementation of the tried and true network virtualization technique. The Linux implementations are extremely lightweight because rather than using a Linux bridge for isolation, they are simply associated with a Linux Ethernet interface or sub-interface to enforce separation between networks and connectivity to the physical network.

MACVLAN offers a number of unique features and capabilities. It has positive performance implications by virtue of having a very simple and lightweight architecture. Rather than port mapping, the MACVLAN driver provides direct access between containers and the physical network. It also allows containers to receive routable IP addresses that are on the subnet of the physical network.

The `macvlan` driver uses the concept of a parent interface. This interface can be a physical interface such as `eth0`, a sub-interface for 802.1q VLAN tagging like `eth0.10` (`.10` representing `VLAN 10`), or even a bonded host adaptor which bundle two Ethernet interfaces into a single logical interface.

A gateway address is required during MACVLAN network configuration. The gateway must be external to the host provided by the network infrastructure. MACVLAN networks allow access between container on the same network. Access between different MACVLAN networks on the same host is not possible without routing outside the host.

![Connecting Containers with a MACVLAN Network](./img/macvlanarch.png)

In this example, we bind a MACVLAN network to `eth0` on the host. We attach two containers to the `mvnet` MACVLAN network and show that they can ping between themselves. Each container has an address on the `192.168.0.0/24` physical network subnet and their default gateway is an interface in the physical network.

```bash
#Creation of MACVLAN network "mvnet" bound to eth0 on the host 
$ docker network create -d macvlan --subnet 192.168.0.0/24 --gateway 192.168.0.1 -o parent=eth0 mvnet

#Creation of containers on the "mvnet" network
$ docker run -itd --name c1 --net mvnet --ip 192.168.0.3 busybox sh
$ docker run -it --name c2 --net mvnet --ip 192.168.0.4 busybox sh
/ # ping 192.168.0.3
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.052 ms
```

As you can see in this diagram, `c1` and `c2` are attached via the MACVLAN network called `macvlan` attached to `eth0` on the host.

### VLAN Trunking with MACVLAN

Trunking 802.1q to a Linux host is notoriously painful for many in operations. It requires configuration file changes in order to be persistent through a reboot. If a bridge is involved, a physical NIC needs to be moved into the bridge, and the bridge then gets the IP address. The `macvlan` driver completely manages sub-interfaces and other components of the MACVLAN network through creation, destruction, and host reboots.

![VLAN Trunking with MACVLAN](./img/trunk-macvlan.png)

When the `macvlan` driver is instantiated with sub-interfaces it allows VLAN trunking to the host and segments containers at L2. The `macvlan` driver automatically creates the sub-interfaces and connects them to the container interfaces. As a result each container will be in a different VLAN, and communication will not be possible between them unless traffic is routed in the physical network. 


```bash
#Creation of  macvlan10 network that will be in VLAN 10
$ docker network create -d macvlan --subnet 192.168.10.0/24 --gateway 192.168.10.1 -o parent=eth0.10macvlan10

#Creation of  macvlan20 network that will be in VLAN 20
$ docker network create -d macvlan --subnet 192.168.20.0/24 --gateway 192.168.20.1 -o parent=eth0.20 macvlan20

#Creation of containers on separate MACVLAN networks
$ docker run -itd --name c1--net macvlan10 --ip 192.168.10.2 busybox sh
$ docker run -it --name c2--net macvlan20 --ip 192.168.20.2 busybox sh
```

In the preceding configuration we've created two separate networks using the `macvlan` driver that are configured to use a sub-interface as their parent interface. The `macvlan` driver creates the sub-interfaces and connects them between the host's `eth0` and the container interfaces. The host interface and upstream switch must be set to `switchport mode trunk` so that VLANs are tagged going across the interface. One or more containers can be connected to a given MACVLAN network to create complex network policies that are segmented via L2.

> Because multiple MAC addresses are living behind a single host interface you might need to enable promiscuous mode on the interface depending on the NIC's support for MAC filtering.  

Next: **[Host (Native) Network Driver](08-host-networking.md)**

## <a name="hostdriver"></a>Host (Native) Network Driver

The `host` network driver connects a container directly to the host networking stack. Containers using the `host` driver reside in the same network namespace as the host itself. Thus, containers will have native bare-metal network performance at the cost of namespace isolation. 

```bash
#Create a container attached to the host network namespace and print its network interfaces
$ docker run -it --net host --name c1 busybox ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:19:5F:BC:F7
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr 08:00:27:85:8E:95
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe85:8e95/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:190780 errors:0 dropped:0 overruns:0 frame:0
          TX packets:58407 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:189367384 (180.5 MiB)  TX bytes:3714724 (3.5 MiB)
...

#Display the interfaces on the host
$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:19:5f:bc:f7
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr 08:00:27:85:8e:95
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe85:8e95/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:190812 errors:0 dropped:0 overruns:0 frame:0
          TX packets:58425 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:189369886 (189.3 MB)  TX bytes:3716346 (3.7 MB)
...
```

In this example we can see that the host and container `c1` share the same interfaces. This has some interesting implications. Traffic passes directly from the container to the host interfaces.

With the `host` driver, Docker does not manage any portion of the container networking stack such as port mapping or routing rules. This means that common networking flags like `-p` and `--icc` have no meaning for the `host` driver. They will be ignored. If the network admin wishes to provide access and policy to containers then this will have to be self-managed on the host or managed by another tool.

Every container using the `host` network will all share the same host interfaces. This makes `host` ill suited for multi-tenant or highly secure applications. `host` containers will have access to every other container on the host. 

Full host access and no automated policy management may make the `host` driver a difficult fit as a general network driver. However, `host` does have some interesting properties that may be applicable for use cases such as ultra high performance applications, troubleshooting, or monitoring.

## <a name="nonedriver"></a>None (Isolated) Network Driver

Similar to the `host` network driver, the `none` network driver is essentially an unmanaged networking option. Docker Engine will not create interfaces inside the container, establish port mapping, or install routes for connectivity. A container using `--net=none` will be completely isolated from other containers and the host. The networking admin or external tools must be responsible for providing this plumbing. In the following example we see that a container using `none` only has a loopback interface and no other interfaces.


```bash
#Create a container using --net=none and display its interfaces 
$ docker run -it --net none busybox ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
 
Unlike the `host` driver, the `none` driver will create a separate namespace for each container. This guarantees container network isolation between any containers and the host. 

 > Containers using `--net=none` or `--net=host` cannot be connected to any other Docker networks.

 Next: **[Physical Network Design Requirements](09-physical-networking.md)**
## <a name="requirements"></a>Physical Network Design Requirements
Docker Datacenter and Docker networking are designed to run over common data center network infrastructure and topologies. Its centralized controller and fault-tolerant cluster guarantee compatibility across a wide range of network environments. The components that provide networking functionality (network provisioning, MAC learning, overlay encryption) are either a part of Docker Engine, UCP, or the Linux kernel itself. No extra components or special networking features are required to run any of the built-in Docker networking drivers.

More specifically, the Docker built-in network drivers have NO requirements for:

- Multicast
- External key-value stores
- Specific routing protocols 
- Layer 2 adjacencies between hosts
- Specific topologies such as spine & leaf, traditional 3-tier, and PoD designs. Any of these topologies are supported.

This is in line with the Container Networking Model which promotes application portability across all environments while still achieving the performance and policy required of applications.

## <a name="sd"></a>Service Discovery Design Considerations

Docker uses embedded DNS to provide service discovery for containers running on a single Docker Engine and `tasks` running in a Docker Swarm. Docker Engine has an internal DNS server that provides name resolution to all of the containers on the host in user-defined bridge, overlay, and MACVLAN networks. Each Docker container ( or `task` in Swarm mode) has a DNS resolver that forwards DNS queries to Docker Engine, which acts as a DNS server. Docker Engine then checks if the DNS query belongs to a container or `service` on network(s) that the requesting container belongs to. If it does, then Docker Engine looks up the IP address that matches a container, `task`, or`service`'s **name** in its key-value store and returns that IP or `service` Virtual IP (VIP) back to the requester. 

Service discovery is _network-scoped_, meaning only containers or tasks that are on the same network can use the embedded DNS functionality. Containers not on the same network cannot resolve each other's addresses. Additionally, only the nodes that have containers or tasks on a particular network store that network's DNS entries. This promotes security and performance.

If the destination container or `service` does not belong on same network(s) as source container, then Docker Engine forwards the DNS query to the configured default DNS server. 

![Service Discovery](./img/DNS.png)

In this example there is a service of two containers called `myservice`. A second service (`client`) exists on the same network. The `client` executes two `curl` operations for `docker.com` and `myservice`. These are the resulting actions:


 - DNS queries are initiated by `client` for `docker.com` and `myservice`.
 - The container's built in resolver intercepts the DNS queries on `127.0.0.11:53` and sends them to Docker Engine's DNS server.
 - `myservice` resolves to the Virtual IP (VIP) of that service which is internally load balanced to the individual task IP addresses. Container names will be resolved as well, albeit directly to their IP address.
 - `docker.com` does not exist as a service name in the `mynet` network and so the request is forwarded to the configured default DNS server.
 
 Next: **[Load Balancing Design Considerations](10-load-balancing.md)**
## <a name="lb"></a>Load Balancing Design Considerations

Load balancing is a major requirement in modern, distributed applications. Docker Swarm mode introduced in 1.12 comes with a native internal and external load balancing functionalities that utilize both `iptables` and `ipvs`, a transport-layer load balancing inside the Linux kernel.

### Internal Load Balancing
When services are created in a Docker Swarm cluster, they are automatically assigned a Virtual IP (VIP) that is part of the service's network. The VIP is returned when resolving the service's name. Traffic to that VIP will be automatically sent to all healthy tasks of that service across the overlay network. This approach avoids any client-side load balancing because only a single IP is returned to the client. Docker takes care of routing and equally distributing the traffic across the healthy service tasks.


![Internal Load Balancing](./img/ipvs.png)

To see the VIP, run a `docker service inspect my_service` as follows:

```
# Create an overlay network called mynet
$ docker network create -d overlay mynet
a59umzkdj2r0ua7x8jxd84dhr

# Create myservice with 2 replicas as part of that network
$ docker service create --network mynet --name myservice --replicas 2 busybox ping localhost
8t5r8cr0f0h6k2c3k7ih4l6f5

# See the VIP that was created for that service
$ docker service inspect myservice
...

"VirtualIPs": [
                {
                    "NetworkID": "a59umzkdj2r0ua7x8jxd84dhr",
                    "Addr": "10.0.0.3/24"
                },
]
              
``` 

> DNS round robin (DNS RR) load balancing is another load balancing option for services (configured with `--endpoint-mode`). In DNS RR mode a VIP is not created for each service. The Docker DNS server resolves a service name to individual container IPs in round robin fashion.


###External Load Balancing (Docker Routing Mesh) 
You can expose services externally by using the `--publish` flag when creating or updating the service. Publishing ports in Docker Swarm mode means that every node in your cluster will be listening on that port. But what happens if the service's task isn't on the node that is listening on that port?

This is where routing mesh comes into play. Routing mesh is a feature introduced in Docker 1.12 that combines `ipvs` and `iptables` to create a powerful cluster-wide transport-layer (L4) load balancer. It allows all the Swarm nodes to accept connections on the services' published ports. When any Swarm node receives traffic destined to the published TCP/UDP port of a running `service`, it forwards it to service's VIP using a pre-defined overlay network called `ingress`. The `ingress` network behaves similarly to other overlay networks but its sole purpose is to transport mesh routing traffic from external clients to cluster services. It uses the same VIP-based internal load balancing as described in the previous section.

Once you launch services, you can create an external DNS record for your applications and map it to any or all Docker Swarm nodes. You do not need to worry about where your container is running as all nodes in your cluster look as one with the routing mesh routing feature.  

```
#Create a service with two replicas and export port 8000 on the cluster
$ docker service create --name app --replicas 2 --network appnet -p 8000:80 nginx
```


![Routing Mess](./img/routing-mesh.png) 

This diagram illustrates how the Routing Mesh works.

- A service is created with two replicas, and it is port mapped externally to port `8000`.
- The routing mesh exposes port `8000` on each host in the cluster.
- Traffic destined for the `app` can enter on any host. In this case the external LB sends the traffic to a host without a service replica.
- The kernel's IPVS load balancer redirects traffic on the `ingress` overlay network to a healthy service replica.

Next: **[Network Security and Encryption Design Considerations](11-security.md)**
## <a name="security"></a>Network Security and Encryption Design Considerations

Network security is a top-of-mind consideration when designing and implementing containerized workloads with Docker. In this section, we will go over three key design considerations that are typically raised around Docker network security and how you can utilize Docker features and best practices to address them. 

### Container Networking Segmentation

Docker allows you to create an isolated network per application using the `overlay` driver. By default different Docker networks are firewalled from eachother. This approach provides a true network isolation at Layer 3. No malicious container can communicate with your application's container unless it's on the same network or your applications' containers expose services on the host port. Therefore, creating networks for each applications adds another layer of security. The principles of "Defense in Depth" still recommends application-level security to protect at L3 and L7.

### Securing the Control Plane

Docker Swarm comes with integrated PKI. All managers and nodes in the Swarm have a cryptographically signed identify in the form of a signed certificate. All manager-to-manager and manager-to-node control communication is secured out of the box with TLS. No need to generate certs externally or set up any CAs manually to get end-to-end control plane traffic secured in Docker Swarm mode. Certificates are periodically and automatically rotated.

### Securing the Data Plane

In Docker Swarm mode the data path (e.g. application traffic) can be encrypted out-of-the-box. This feature uses IPSec tunnels to encrypt network traffic as it leaves the source container and decrypts it as it enters the destination container.  This ensure that your application traffic is highly secure when it's in transit regardless of the underlying networks. In a hybrid, multi-tenant, or multi-cloud environment, it is crucial to ensure data is secure as it traverses networks you might not have control over. 

This diagram illustrates how to secure communication between two containers running on different hosts in a Docker Swarm. 

![Secure Communications between 2 Containers on Different Hosts](img/ipsec.png)

This feature works with the `overlay` driver in Swarm mode only and can be enabled per network at the time of creation by adding the `--opt encrypted=true` option (e.g `docker network create -d overlay --opt encrypted=true <NETWORK_NAME>`). After the network gets created, you can launch services on that network (e.g `docker service create --network <NETWORK_NAME> <IMAGE> <COMMAND>`). When two tasks of the same services are created on two different hosts, an IPsec tunnel is created between them and traffic gets encrypted as it leaves the source host and gets decrypted as it enters the destination host. 

The Swarm leader periodically regenerates a symmetrical key and distributes it securely to all cluster nodes. This key is used by IPsec to encrypt and decrypt data plane traffic. The encryption is implemented via IPSec in host-to-host transport mode using AES-GCM.

Next: **[IP Address Management](12-ipaddress-management.md)**
## <a name="ipam"></a>IP Address Management

The Container Networking Model (CNM) provides flexibility in how IP addresses are managed. There are two methods for IP address management.

- CNM has a built-in IPAM driver that does simple allocation of IP addresses globally for a cluster and prevents overlapping allocations. The built-in IPAM driver is what is used by default if no other driver is specified. 
- CNM has interfaces to use plug-in IPAM drivers from other vendors and the community. These drivers can provide integration into existing vendor or self-built IPAM tools. 

Manual configuration of container IP addresses and network subnets can be done using UCP, the CLI, or Docker APIs. The address request will go through the chosen driver which will decide how to process the request.

Subnet size and design is largely dependent on a given application and the specific network driver. IP address space design is covered in more depth for each [Network Deployment Model](#models) in the next section. The uses of port mapping, overlays, and MACVLAN all have implications on how IP addressing is arranged. In general, container addressing falls into two buckets. Internal container networks (bridge and overlay) address containers with IP addresses that are not routable on the physical network by default. MACVLAN networks provide IP addresses to containers that are on the subnet of the physical network. Thus, traffic from container interfaces can be routable on the physical network. It is important to note that subnets for internal networks (bridge, overlay) should not conflict with the IP space of the physical underlay network. Overlapping address space can cause traffic to not reach its destination. 

Next: **[Network Troubleshooting](13-troubleshooting.md)**
## <a name="tshoot"></a>Network Troubleshooting

Docker network troubleshooting can be difficult for devops and network engineers. With proper understanding of how Docker networking works and the right set of tools, you can troubleshoot and resolve these network issues. One recommended way is to use the [netshoot](https://github.com/nicolaka/netshoot) container to troubleshoot network problems. The `netshoot` container has a set of powerful networking troubleshooting tools that can be used to troubleshoot Docker network issues.  

Next: **[Network Deployment Models](14-network-models.md)**
## <a name="models"></a>Network Deployment Models
Docker Engine and community provide multiple drivers to use. These drivers can be configured in multiple ways, and the physical network design and configuration will also affect network behavior. This section looks at different configurations and how they interoperate with the application and the physical network. This is not an exhaustive list but a description of common methods of deployment.

![Common Methods of Network Deployment](./img/driver-comparison.png)

Back to [Concepts](README.md)
or
On to [Tutorials](../tutorials.md)
