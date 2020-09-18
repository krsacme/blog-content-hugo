---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2018-06-20T09:00:00+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- openstack
- tripleo
- dpdk
- ovs
- vhost
title: 'OVS-DPDK: Vhostuser socket Mode'
type: post
---

In the newton release, the default vhostuser mode in OvS is `dpdkvhostuser`.
And from ocata onwards, the default mode in the neutron has been changed to
`dpdkvhostuserclient` mode. This post provides the information on vhostuser
migration and verifying the vhostuser modes of the VMs created with
`dpdkvhostuser` mode.

In order to understand the difference between the two modes and the advantage
of moving to `dpdkvhostuserclient` mode, read the OvS documentation on
[vhostuser] modes. In short, vhostuser allows Qemu to fetch/put network data
to OvS-DPDK without overloading Qemu with the translation. And the vhostuser
socket is a UNIX domain socket, created to establish the communication between
Qemu and OvS-DPDK. This communication follows a specific messaging format
detailed in the [Qemu's vhost user] document. In the `dpdkvhostuserclient`
mode, Qemu acts as the server and creates the vhostuser socket. And OvS acts
as client which connects to the created socket. Using this mode,
crashes/restarts on the OvS-DPDK application does not require to restart the
VM, as the client will establish the connection after starting.

The steps for migration existing VMs with server to client mode in OvS is
detailed in the [minor update] documentation, along with the limitations. Lets
see how and where to check the vhostuser mode information in both Qemu and
OvS. Following sections shows the difference that these two modes creates on a
compute node in a TripleO deployment. This is applicable from newton through
queens.

### dpdkvhostuser

In the `dpdkvhostuser` mode, the socket will be created  by OvS in the
`/var/run/openvswitch` directory. The OvS service file should be patched to
make OvS run with `qemu` as group ownership. This allows the Qemu, which is
the client, to connect to the socket.

This snippet shows that the vhostuser socket is created with the ownership as
`root:qemu` in the default directory.
```
$ ll /var/run/openvswitch/vhu1879c31a-fa
srwxrwxr-x. 1 root qemu 0 Jun 19 04:38 /var/run/openvswitch/vhu1879c31a-fa
```

This snippet shows that the vhostuser socket's port details attached to the
`br-int` bridge, which also shows the mode of the socket.
```
# Partial output
$ ovs-vstl show
    Bridge br-int
        Port "vhu1879c31a-fa"
            tag: 1
            Interface "vhu1879c31a-fa"
                type: dpdkvhostuser
```

The OvS process runs with the patched ownership. This snippet shows that the
`ovs-vswitchd` process runs with the `root:qemu` ownership.
```
# Partial output
$ ps ax -o user,group,pid,command,args | grep -e ovsdb-server -e ovs-vswitchd
root     root       11770 ovsdb-server /etc/openvswit
root     qemu       11806 ovs-vswitchd unix:/var/run/
```

This snippet shows the extract from the virsh xml of a VM with vhostuser
socket is connected in the client mode.
```
# Partial output
$ virsh dumpxml instance-00000002
    <interface type='vhostuser'>
      <mac address='fa:16:3e:4b:ea:c4'/>
      <source type='unix' path='/var/run/openvswitch/vhu1879c31a-fa' mode='client'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

The libvirt logs shows the socket and its mode attached to the VM. If there is
any failure in the migration, this log will help to understand the mode of the
vhostuser socket attached to the VM.
```
# Log from /var/log/libvirt/qemu/instance-00000002.log file
-chardev socket,id=charnet0,path=/var/run/openvswitch/vhueacd1f04-7a
```

The process associated with the VM runs with `qemu:qemu` ownership.
```
# Partial output
$ ps ax -o user,group,pid,command,args | grep -e qemu-kvm
qemu     qemu      166572 /usr/libexec/qemu-kvm -name /usr/libexec/qemu-kvm -name guest=instance-00000002,debug-threads=on
```

### dpdkvhostuserclient
In the `dpdkvhostuserclient` mode, the socket will be created  by Qemu in the
`/var/lib/vhost_sockets` directory. A new group `hugetlbfs` has been created
which will be used to share the socket between Qemu and OvS. The `group` value
in the `/etc/libvirt/qemu.conf` will be configured with the new group
`hugetlbfs`. All the VMs created then on will run with `qemu:hugetlbfs`
ownership. OvS will run with the ownership `openvswitch:hugetlbfs`, which has
been introduced from OvS2.8 version onwards. The configuration `OVS_USER_ID`
in the file `/etc/sysconfig/openvswitch`, will configure the ownership of the
OvS process that will run on the node.

The `ovsdb-server` service file will change the ownership of the
`/var/run/openvswitch` and `ovs-vswitchd` service file will change the
ownership of `/dev/hugepages` to the value configured using `OVS_USER_ID`.

Below snippets demonstrate this behavior of change in the ownership of the
files and process and other configuration compare to the older vhostuser mode.

```
$ ll /var/lib/vhost_sockets/vhu28b691d4-0e
srwxrwxr-x. 1 qemu hugetlbfs 0 Jun 21 12:00 /var/lib/vhost_sockets/vhu28b691d4-0e
```

```
# Partial output
$ ovs-vstl show
        Port "vhu28b691d4-0e"
            tag: 1
            Interface "vhu28b691d4-0e"
                type: dpdkvhostuserclient
                options: {vhost-server-path="/var/lib/vhost_sockets/vhu28b691d4-0e"}
```

```
# Partial output
$ ps ax -o user,group,pid,command,args | grep -e ovsdb-server -e ovs-vswitchd
openvsw+ hugetlb+    1691 ovsdb-server /etc/openvswit ovsdb-server
openvsw+ hugetlb+    1773 ovs-vswitchd unix:/var/run/ ovs-vswitchd
```

This snippet shows the extract from the virsh xml of a VM with Qemu as server.

```
# Partial output
$ virsh dumpxml instance-00000003
    <interface type='vhostuser'>
      <mac address='fa:16:3e:6d:49:e8'/>
      <source type='unix' path='/var/lib/vhost_sockets/vhu28b691d4-0e' mode='server'/>
      <target dev='vhu28b691d4-0e'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

```
# Log from /var/log/libvirt/qemu/instance-00000003.log file
-chardev socket,id=charnet0,path=/var/lib/vhost_sockets/vhu28b691d4-0e,server
```

```
# Partial output
$ ps ax -o user,group,pid,command,args | grep -e qemu-kvm
qemu     hugetlb+  152317 /usr/libexec/qemu-kvm -name /usr/libexec/qemu-kvm -name guest=instance-00000003,debug-threads=on
```

### Notes

* In newton based deployment, all the openstack services are deployed as
  systemd services, where as on queens based deployment, all the openstack
  services will be deployed using the containers. But still openvswitch runs
  as the systemd service, which requires the vhostuser socket created by Qemu
  (running as nova_libvirt container) to share the same group id with the OvS
  running the host. In order to facilitate the sharing of vhostuser between
  host and kolla based nova_libvirt container, the gid of `hugetlbfs` in the
  host has been aligned with the gid value of the `hugetlbfs` in the kolla
  container. Also this allows us to enable the newton based deployment to
  upgrade to queens without much difficulty. The gid value of the `hugetlbfs`
  group is configured as 42477.


[vhostuser]: http://ovs-reviews.readthedocs.io/en/latest/topics/dpdk/vhost-user.html#vhost-user-vs-vhost-user-client
[minor update]: /post/newton-minor-update/#existing-vms-and-migration
[Qemu's vhost user]: https://github.com/qemu/qemu/blob/master/docs/interop/vhost-user.txt
