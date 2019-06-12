- [Diagram](#diagram)
- [Objectives](#objectives)
- [Requirements](#requirements)
  - [IPv6 network](#ipv6-network)
- [Hypervisor](#hypervisor)
  - [Bridges](#bridges)
- [Firewall (pfSense)](#firewall-pfsense)
  - [Interfaces assignments in pfSense](#interfaces-assignments-in-pfsense)
    - [IPv4](#ipv4)
    - [IPv6](#ipv6)
  - [Routing in pfSense](#routing-in-pfsense)
  - [MAC assignment for IPv6](#mac-assignment-for-ipv6)
  - [Firewall Rules](#firewall-rules)
- [VM configuration](#vm-configuration)
  - [Docker configuration](#docker-configuration)
## Diagram
![Diagram](images/Docker&#32;IPv6&#32;Securely.png)
## Objectives 
* Automatic provisioning of IPv6
   * Each Docker container will receive an IPv6 automatically.
* Routed IPv6 traffic between container in different Hosts
   * Each host will have a /80 that will be routed trough pfSense 
* Secured public access to our containers via IPv6.
   * They will be reachable publicly so we want to be able to whitelist open ports in a secure way.
* Out of band Firewall
   * Having an out of band firewall, meaning outside of the VM, will increase the security of the system
   * 
## Requirements
### IPv6 network
For this article, we will use a /64 IPv6 network because its what our Hosting provider (Hetzner) gives us.
This will be divided in 4096 /76s:
* z: x:y:w:10::1/76 WAN in Pfsense
* z: x:y:w:20::1/76 VIP (or WAN6) gateway for VM1
  * z: x:y:w:21::/80 for Docker containers in VM1
* z: x:y:w::30:1/76 VIP gateway for VM2
  * z: x:y:w:31::1/80 for Docker containers in VM2
* z: x:y:w:3e80::1/76 VMx
  * z: x:y:w:3e81::1/80 for Docker containers in VMx

## Hypervisor
We will need a way to provision VMs, for this article we selected Proxmox
More information: 
* https://www.proxmox.com/en/proxmox-ve/get-started
* How to install it remotely on any server: https://www.linkedin.com/pulse/installing-any-os-headless-server-rodrigo-leven/

![Proxmox](images/proxmox.png)


### Bridges
We will create 3 bridges:
* **vmbr0**: It will connect to the internet and receive both IPv4 and IPv6 traffic but will only have the main IPv4 assigned xxx.yyy.145.162/26
* **vmbr1**: The internal network shared between VMs, 10.x.x.x
* **vmbr2**: This bridge will hold the IPv6 network for our VMs, in this case a /64

![Bridges](images/bridges.png)
## Firewall (pfSense)
We will need an out of band Firewall to be able to whitelist open ports and for this, we are going to use pfSense.
 
More information here: https://docs.netgate.com/pfsense/en/latest/virtualization/virtualizing-pfsense-with-proxmox.html

We will add 3 network cards and configure each one to one of the Bridges we created before

![pfSense](images/pfsense.png)

### Interfaces assignments in pfSense
![pfSense](images/pfsense_interfaces2.png)
#### IPv4
* WAN NAT IPv4: xxx.yyy.145.136/26
* WAN NAT IPv6: z: x:y:w:10::1/76
* LAN 10.10.10.2

We have 2 IPv4 assigned , one to access the Host KVM and other to use as NAT and the internal LAN.
#### IPv6
Default gateways for the VM hosts.
* WAN6 z: x:y:w:20::1
  * VIP z: x:y:w:30::1
  ...
  * VIP z: x:y:w:3e80::1


### Routing in pfSense

The default GW for IPv6 is fe80::1 trough the WAN interface (not WAN6) this is because this interface is connected to the KVM bridge that has access to the connection to the internet.

Each Docker network in the VM host gets a static route so they can comunicate between each other.
For this we need to define a Gateway  as z:x:y:w:20::2 trough the WAN6 interface called "vm1" in the picture.

![pfSense bridge](images/pfsense_gateways.png)

Finally we add a static route saying that our container subnet 21::/80 can be reached trough the "vm1" gateway that means trough the interface WAN6 via the host at 20::2.

![pfSense bridge](images/pfsense_static_routes.png)


### MAC assignment for IPv6

Our hosting provider requires that our IPv6 traffic comes from the same MAC address associated with one of our IPv4, in this case its the extra NAT IPv4.

![pfSense bridge](images/hetzner_ipv6.png)


### Firewall Rules
We need to create the following firewall rules:
* **WAN:**
  * ANY to our IPv6 /80
  * ANY to our IPv6/ports we want to be public
  * From WAN6 to ANY (outgoing traffic)

* **WAN6:**
  * Protocol IPv6 to WAN6 (outgoing traffic)

## VM configuration
In this example we use systemd-networkd but the same idea can be used on any network manager.
https://wiki.archlinux.org/index.php/Systemd-networkd

Notice we use ::2 for our VM IPv6 and WAN6 interface IP as a default route.

```
[Match]
Name=eth1

[Network]
Address=z:x:y:w:20::2/80
Gateway=z:x:y:w:20::1

[Route]
Gateway=z:x:y:w:20::1

```

This will give us:

```
# ip -6 addr show dev eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master docker0 state UP group default qlen 1000
    inet6 z:x:y:w:20::2/76 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::e470:c4ff:fe34:1491/64 scope link 
       valid_lft forever preferred_lft forever


```

Test we have IPv6 connectivity:
```
# ping -6 www.google.com
PING www.google.com(ams16s29-in-x04.1e100.net (2a00:1450:400e:804::2004)) 56 data bytes
64 bytes from ams16s29-in-x04.1e100.net (2a00:1450:400e:804::2004): icmp_seq=1 ttl=55 time=32.2 ms
64 bytes from ams16s29-in-x04.1e100.net (2a00:1450:400e:804::2004): icmp_seq=2 ttl=55 time=32.5 ms
64 bytes from ams16s29-in-x04.1e100.net (2a00:1450:400e:804::2004): icmp_seq=3 ttl=55 time=32.4 ms
```

### Docker configuration
More information: https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#routed-network-environment

We configure docker to use our /80 subnets under 21::/80.

```
# cat /etc/docker/daemon.json 
{
  "ipv6": true,
  "fixed-cidr-v6": "z:x:y:w:21::/80"
}
```
Test it by using an alpine container:

```
# docker run -it alpine ash
/ # ip -6 route
z:x:y:w:21::/80 dev eth0  metric 256 
fe80::/64 dev eth0  metric 256 
default via z:x:y:w:21::1 dev eth0  metric 1024 
ff00::/8 dev eth0  metric 256 

/ # ip -6 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
43: eth0@if44: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 state UP 
    inet6 z:x:y:w:21:242:ac11:2/80 scope global flags 02 
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link 
       valid_lft forever preferred_lft forever

/ # ping -6 www.google.com
PING www.google.com (2a00:1450:400e:807::2004): 56 data bytes
64 bytes from 2a00:1450:400e:807::2004: seq=0 ttl=55 time=26.828 ms
64 bytes from 2a00:1450:400e:807::2004: seq=1 ttl=55 time=26.904 ms

```

From here we can see we got z:.x:y:w:21:242:ac11:2/80 and ping to google works.

