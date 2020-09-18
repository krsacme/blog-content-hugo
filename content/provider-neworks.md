---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2017-06-05T13:18:22+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- openstack
- provider
- network
title: OpenStack Provider Networks
type: post
---

As an OpenStack newcomer, I got confused around the concept of "Provider
Network" on a OpenStack cluster. As many of the core component developers does
not even know or came acorss such a term, it is always a challege to
understand and work with it. In a production deployment of any type, may be an
Enterprise deployment or NFV deploymnet, "Provider Network" is an integral
part of it. Any consultant who works on a production deployment, will
eventually come across it. Here I am posting my understanding of Provider
Networks, as I understood it from experts.
<!--more-->

## What is a "Provider Network"?
  * Provider network is external to the OpenStack cluster, which should be
    accessible to the guest VMs.
  * This network would be configured by the cloud operator and cannot be
    controlled by the tenant.
  * It is of type either *flat* or *VLAN*. It does not support *VxLAN* network
    type.
  * The propeties of the provide network like physical bridge, VLAN
    segmentation id are choosed by the cloud operator, rather than neutron.
    All these properties will be provided to ``neutron net-create`` command to
    create a provider network.
  * It is possible to configure the provider network to acquire address via
    DHCP running externally or via neutron.

Following command is used to create a flat provider network:
```bash

neutron net-create dpdkflat --provider:network_type flat --provider:physical_network dpdknet
neutron subnet-create dpdkflat 192.0.9.0/24 --name dpdkflat

```

Following command is used to create a VLAN provider network:
```bash

neutron net-create dpdkvlan --provider:network_type vlan --provider:segmentation_id 100 --provider:physical_network dpdknet
neutron subnet-create dpdkvlan 192.0.10.0/24 --name dpdkvlan

```
