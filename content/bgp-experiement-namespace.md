---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2021-11-09T09:00:00+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- bgp
- frr
title: BGP Experiement - Network Namespaces
type: post
---

This blog is about an expriement to familiarize with how BGP works and
understand the FRR configurations for BGP. In this, 2 network namepsaces
(`blue` and `red`) are created with each having a FRR instance running within
the namespace. Both network namepsace are connected via veth pair so BGP can
communicate their respective routes. Network namepsace has local interface
created as macvlan with local IP. We can observe that the local routes is
transmitted to the neighbour namepsace. The blog will explain the steps to
create the namespace based FRR setup and how to configure and confirm the
routes update is working.

### Creating Namespace

Let us create 2 namepsaces `red` and `blue`. It will als setup a veth pair
betwen them.

```
ip netns add red
ip netns add blue
ip link add veth0 netns red type veth peer name veth1 netns blue
ip netns exec red  ip l s dev veth0 up
ip netns exec blue ip l s dev veth1 up

ip netns exec red  ip a a dev veth0 192.168.10.25/24
ip netns exec blue ip a a dev veth1 192.168.10.5/24
```

Create a local interface for local to each namespace whole route should be
advertised to its nighbour.

```
ip netns exec red ip link add macvlan1 link veth0 type macvlan mode bridge
ip netns exec red ip link set dev macvlan1 up
ip netns exec red ip addr add 192.168.40.5/24 dev macvlan1

ip netns exec blue ip link add macvlan2 link veth1 type macvlan mode bridge
ip netns exec blue ip link set dev macvlan2 up
ip netns exec blue ip addr add 192.168.30.5/24 dev macvlan2
```

### Create FRR Configuration

Create a configuration director for `blue` namespace and configure the
required files. Also enable enable BGP in the `daemons` file by following the
regular FRR documentation. And the follow the same approach for `red`
namepsace too.

```
mkdir /etc/frr/blue
cp /etc/frr/daemons /etc/frr/blue/

mkdir /etc/frr/red
cp /etc/frr/daemons /etc/frr/red/
```

#### FRR conf - Blue

Lets create `blue` namespace configuration for frr at `/etc/frr/blue/frr.conf`
with below content.

```
log file /var/log/frr/blue.log debugging
!
router bgp 20
bgp default ipv4-unicast
no bgp default ipv6-unicast
network 192.168.30.0/24
neighbor 192.168.10.25 remote-as 21
neighbor 192.168.10.25 soft-reconfiguration inbound
neighbor 192.168.10.25 disable-connected-check
address-family ipv4 unicast
 redistribute connected
 neighbor 192.168.10.25 route-map RMAP in
 neighbor 192.168.10.25 route-map RMAP out
exit-address-family
!
ip prefix-list PLIST permit 192.168.0.0/16 le 32 ge 16
!
route-map RMAP permit 21
match ip address prefix-list PLIST
set community 21:80
```

#### FRR conf - Red

Lets create `red` namespace configuration for frr at `/etc/frr/red/frr.conf`
with below content.

```
log file /var/log/frr/red.log debugging
!
router bgp 21
bgp default ipv4-unicast
no bgp default ipv6-unicast
network 192.168.40.0/24
neighbor 192.168.10.5 remote-as 20
neighbor 192.168.10.5 soft-reconfiguration inbound
neighbor 192.168.10.5 disable-connected-check
address-family ipv4 unicast
 redistribute connected
 neighbor 192.168.10.5 route-map RMAP in
 neighbor 192.168.10.5 route-map RMAP out
exit-address-family
!
route-map RMAP permit 20
set community 20:80
```

### Start FRR in namespaces

Lets start FRR in respective namespaces using below command:

```
ip netns exec blue  /usr/lib/frr/frrinit.sh start blue
ip netns exec red  /usr/lib/frr/frrinit.sh start red
```

Observe with the command `ps aux | grep bgp` that two instance of FRR BGP is
running with respecive namespace specific configuration. After a while, the
local routes should be transmitted to the neighbour so that the local IP of
`blue` namespace is reacheable from `red` namespace.


## Validation

Login to FRR using `vtysh` to the specific instance of FRR running on `blue`
namespace using below command:

```
ip netns exec blue  /usr/bin/vtysh --vty_socket /var/run/frr/blue/
```

Once inside the vtysh shell, run the command `show ip route` to diplay all the
applied routes in FRR. In the list. Observe that remote route
`192.168.40.0/24` (of `red` namepsace) is listed with prefix `B` which
signifies that it is received via BGP.

```
fedora# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

C>* 192.168.10.0/24 is directly connected, veth1, 17:34:19
C>* 192.168.30.0/24 is directly connected, macvlan2, 17:34:19
B>* 192.168.40.0/24 [20/0] via 192.168.10.25, veth1, weight 1, 16:47:21
```

List the routes in the `blue` namespace to observe that the received route is
applied to kernel.

```
[root@fedora ~]# ip netns exec blue ip r
192.168.10.0/24 dev veth1 proto kernel scope link src 192.168.10.5
192.168.30.0/24 dev macvlan2 proto kernel scope link src 192.168.30.5
192.168.40.0/24 nhid 16 via 192.168.10.25 dev veth1 proto bgp metric 20
```

Confirm that the remove IP `192.168.40.5` is reacheable from blue namepsace
with ping.

```
[root@fedora ~]# ip netns exec blue ping -c 1 192.168.40.5
PING 192.168.40.5 (192.168.40.5) 56(84) bytes of data.
64 bytes from 192.168.40.5: icmp_seq=1 ttl=64 time=0.116 ms

--- 192.168.40.5 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.116/0.116/0.116/0.000 ms
```

## Explanation

Lets get into the details of one side of the configuration to understand it
bit deeper. Lets focus on `blue` namespace. It is configured with IP
`192.168.10.5` on the veth par interface `veth1` where the other end `veth0`
is connected with `red` namespace with IP `192.168.10.25`.

`blue` network's ASN is 20 and `red` network's ASN is 21. In order to
configure `red` network as a neighbour for `blue` network, the frr.conf file
of `bule` network has the neighbour configuration as `neighbor 192.168.10.25
remote-as 21`. This implies that the `red` network with ASN 21 is neighbour
to `blue` network.

In order to advertise a local route, the network information has to be added
to the conf file. In case of `blue` network, the local network is
`192.168.30.0/24` with IP of `macvlan2` interface configured as
`192.168.30.5`. In order to advertise this network, add `network
192.168.30.0/24` to the conf file.

In order to receive a remote local route, the neighbour namepsace `red` is
configure with a local network `192.168.40.0/24` which will be advertised to
`blue` network. Our aim is to be able to ping the remote local IP
`192.168.40.5` of `red` namespace from `blue` namepsace, without manual route
changes.

It is good to enable `soft-reconfiguration inbound`, which allows to store the
received routes in memory so that the command `show bgp ipv4 uni neighbor
192.168.10.25 received-routes` can show the received routes even if the route
is rejected because of other reasons. It is good for debugging issues. The
`soft-reconfiguration` is useful to change policies without resetting the BGP
session. The received routes will be stored and will be reapplied with the
policy change.

`route-map` has to be configure to allow input and output routes to be enabled
to receive and advertise routes to a neighbour. Additionally, the `route-map`
can also be configure with additional filtering options to restrict what IP
prefixes are to be advertised for output and what IP prefixes to be received
for input route map. By default all prefixes are allowed if it is not
specified. In our example, `red` network's frr configuration does not have
any prefixes, where as `blue` neworks configuration have prefixes added to
allow both local and remote local routes.
