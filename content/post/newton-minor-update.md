+++
date = "2018-06-20T09:00:00+05:30"
title = "Newton: Minor Update (OVS-DPDK) - OvS2.9"
tags = ["openstack","tripleo", "minor", "update", "newton", "dpdk"]
highlight = true
math = false

[header]
  caption = ""
  image = ""

+++

Newton (OSP10) has seen a variety of OpenvSwitch version supported. Initial
release was with OvS2.5 and later it has been updated to OvS2.6, which is
present for a long time of the support cycle. Recently, in the time-line with
with queens (OSP13) release, OvS2.9 version is planned to be supported. In
order to facilitate FFU (Fast Forward Upgrades) from newton to queens, bringing
the support of OvS2.9 to newton (OSP10) will help in reduce the cluster
downtime for the upgrade. This post details on the steps and major changes
along with known issues.

### Mapping OpenvSwitch version with TripleO
As you might be aware that there is no internal mapping for the incremental
releases of newton (OSP10), it would be hard to map OvS version with TripleO.
This table provides the OvS version agains the tripleo-heat-templates RPM
version, which are released together.
<table class="table"> <thead>
<tr><td>Date</td><td>OpenvSwitch Version</td><td>tripleo-heat-templates
version</td></tr> </thead> <tbody>
<tr><td>12-Dec-2016</td><td>2.5.0-14</td><td>5.1.0-7</td></tr>
<tr><td>19-Jun-2017</td><td>2.6.1-10</td><td>5.2.0-20</td></tr>
<tr><td>29-Jun-2018</td><td>2.9.0-19</td><td>5.3.10-7</td></tr> </tbody>
</table>


### Highlights of release

* Newton release with OvS2.6 has the default vhost user mode as
  `dpdkvhostuser`, in which OpenvSwitch acts as server and Qemu acts as
  client. But with this release along with OvS2.9, the default mode has been
  changed in neutron to `dpdkvhostuserclient`, in which OpenvSwitch acts as
  client and Qemu acts as server. It provides the advantage of network
  reconnecting when OpenvSwitch is restarted (because of crash) without
  restart the guest VM.

* Newton release with OvS2.6 is deployed by modifying the OpenvSwitch service
  file to make it to run with group as `qemu`. With OvS2.9, this workaround of
  modifying OpenvSwitch service file is not required, as both OpenvSwitch and
  qemu will be configured to run with same group as `hugetlbfs`. And the vhost
  sockets created by qemu on the directory `/var/lib/vhost_sockets` will be
  accessible to OpenvSwitch running with this group.

### Minor Update steps in Newton

Assuming that newton (OSP10) with OvS2.6 is deployed successfully as per the
documentation and the undercloud is updated to the latest newton release, then
here is the procedure to update the overcloud nodes with OVS-DPDK deployment:

1. Configure the vhostuser socket directory for the `dpdkvhostuserclient`
   mode. The existing directory `/var/run/openvswitch` does not have group
   write access needed for qemu to create the sockets. So, instead of exposing
   other files to the common group, the creation of vhostuser sockets has been
   moved to a new directory. Create an environment file `ovs-dpdk-update.yaml`
   with below contents:

    ```yaml
      parameter_defaults:
        NeutronVhostuserSocketDir: '/var/lib/vhost_sockets'
    ```

2. Add a new environment file `environments/ovs-dpdk-permissions.yaml` for the
   deploy command to update the plan before starting the minor update. This
   environment file provides the new group `hugetlbfs` value to the parameter
   `VhostuserSocketGroup`, which will apply the group to the
   `/etc/libvirt/qemu.conf` to create qemu VMs with group as `hugetlbfs`. And
   additionally create the new vhostuser socket directory with the specified
   permission.

     **_Note:_** On a cluster with OVS-DPDK deployment, all the compute nodes
     will be configured with `hugetlbfs` for qemu to avoid confusion and allow
     easy adaptability of services in the existing nodes.

3. Execute the plan update command with the above mentioned changes. Deploy
   command with option `--update-plan-only` will upload the latest templates
   after undercloud update to the swift container for the minor update to use.
   And also uploads the user provided additional environment files so that it
   can be added to the deployment. Once the plan is uploaded and rendered with
   Jinja2, the deploy command stops to allow the minor update command to
   continue.

    ```bash
    openstack overcloud deploy --templates \
        << include all the existing files >> \
        -e /usr/share/openstack-tripleo-heat-templates/environments/ovs-dpdk-permissions.yaml \
        -e ovs-dpdk-update.yaml \
        --update-plan-only
    ```

4. Configure the overcloud nodes have the correct newton (OSP10) repos
   configure to start the package update process.

5. Execute minor update which starts with the package update on all the
   overcloud nodes, followed by running the puppet step configuration.

    ```bash
        yes "" | openstack overcloud update stack -i overcloud
    ```

### Existing VMs and migration

Once the update is completed, it is required to restart the overcloud nodes to
use the updated kernel and openvswitch. Restarting the overcloud nodes and
starting the VM again will start the VM in the `dpdkvhostuser` mode only. In
order to migrate to the `dpdkvhostuserclient` mode, the VMs has to be either
cold-migrated or re-created with a snapshot image of the existing VM.

**_Note_**: Live migration will fail with this forced mode changed. During the
live migration with this change, nova will not re-create the virsh xml, but
will try to use the same as existing mode (Qemu in client mode). But
openvswitch and neutron will migrate the existing port to the new mode (OvS
from server mode to client mode). This results in failure as both OvS and Qemu
are in client mode.

The existing VM can be migrated in place before a reboot or cold migrated to
another host (which has been restarted already), but both the migrations
involve a downtime.

#### 1. Migrate in-place with snapshot

The existing VMs with `dpdkvhostuser` mode can be migrated to client mode in-
place without rebooting any of the nodes. As OvS2.6 has the support for
`dpdkvhostuserclient` mode, it is possible to re-create the instance in-place
with OvS2.6 itself after minor update but before reboot. This method of
migration works when the node is rebooted to OvS2.9 also. If in-place migration
is mandatory, ensure the nova services are disabled on other nodes or other
means of controlling the scheduler, so that it will be created on the same
node.

```bash
# Stop the instance and create a snapshot image
openstack server stop <instance>
openstack server image create --name dpdk_vm1_migration_snapshot <instance>

# Ensure the snapshot image is listed
openstack image list

# Delete the old instance so that the resources available to re-create the VM
openstack server delete <instance>

openstack server create --flavor m1.nano --nic net-id=<dpdk_net_id> \
    --image dpdk_vm1_migration_snapshot --key-name <key_name> <new_instance>
```

#### 2. Cold Migration

Cold migration of the VM also works to migrate the instance, but it has to be
migrated to another node. So it is better to reboot the target node, so that
kernel and OvS update takes effect before cold migrate.

```bash
openstack server migrate <instance> --block-migration --wait
```