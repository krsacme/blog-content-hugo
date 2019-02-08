+++
type = "post"
date = "2019-02-08T09:00:00+05:30"
title = "Fast Forward Upgrades with OvS-DPDK"
tags = ["tripleo", "dpdk", "ffu"]
highlight = true
math = false
author = "Saravanan KR"

[header]
  caption = ""
  image = ""

+++

For OvS-DPDK deployments, FFU upgrade from newton (OSP10) to queens (OSP13)
requires extra care on migrating the templates. Undercloud upgrade procedure
does not involve NFV specific cases, so following the documented procedure
should be enough. Below sections provides the changes required in addition to
the general FFU procedure.

## Using Custom Role for OvS-DPDK

This section details the changes required on the templates, when a custom role
is used for OvS-DPDK, instead of the default `Compute` role.

### Limitation of Newton

In newton, the OvS-DPDK service definition is in
`puppet/services/neutron-ovs-dpdk-agent.yaml` file. And in queens, with the
implementation of containers, the containerized version of the Ovs-DPDK
service definition has to be used, which is available in file
`docker/services/neutron-ovs-dpdk-agent.yaml`.

In newton, the OvS-DPDK service definition file is mapped to the existing
TripleO service registry `OS::TripleO::Services::ComputeNeutronOvsAgent`.
As the existing service registry (of regular OvS agent) is used for DPDK, in
order to deploy a cluster with both Compute and ComputeOvsDpdk roles, the
existing environment file `environments/neutron-ovs-dpdk.yaml` could not
be used.

Deployer have to create a custom TripleO service registry in the user
environment file. The same registry mapping has to be maintained for the FFU
upgrade and any other options after the upgrade too. The same
`roles_data.yaml` file has to be maintained along with user's defined
OvS-DPDK service registry. A sample user registry environment could look like:

```yaml
resource_registry:
  'OS::TripleO::Services::ComputeNeutronOvsDpdkAgent':
    /usr/share/openstack-tripleo-heat-templates/puppet/services/neutron-ovs-dpdk-agent.yaml
```

In the above user's environment file, a new registry has been added as
`OS::TripleO::Services::ComputeNeutronOvsDpdkAgent`, which has to be added
to the custom role, on which DPDK should be enabled (like
`ComputeOvsDpdk`).


### Enhancements in Queens

In order to avoid the user registry mapping, in queens, a new TripleO service
registry mapping `OS::TripleO::Services::ComputeNeutronOvsDpdk` has been
introduced, which as been added to the new role definitions of ComputeOvsDpdk
and ComputeOvsDpdkSriov. And the container service definition has been added
to the existing environment file, mapped to the new service.

This change is backward in-compatible and it requires upgrades to maintain the
same old service registry, but it requires to update to the containerized
version of the service definition. For FFU upgrades, if the user has used a
custom registry like `ComputeNeutronOvsDpdkAgent`, the same environment file
has to be updated like below:

```yaml
resource_registry:
  'OS::TripleO::Services::ComputeNeutronOvsDpdkAgent':
    /usr/share/openstack-tripleo-heat-templates/docker/services/neutron-ovs-dpdk-agent.yaml
```

## Using `Compute` Role for OvS-DPDK

This section explains the changes required when OvS-DPDK is enabled on the
existing `Compute` role itself.

It is also possible to enable OvS-DPDK on the default `Compute` role itself,
by using the provided environment file (`neutron-ovs-dpdk.yaml`). During the
upgrade, the same environment file has been updated to the new service
registry definition instead of `ComputeNeutronOvsAgent`. So, using the default
roles_data file with this environment file will not enable OvS-DPDK. In order
to fix this, an user environment file with the required registry mapping as
like newton has to be added (to the containerized service):

```yaml
resource_registry:
  'OS::TripleO::Services::ComputeNeutronOvsAgent':
    /usr/share/openstack-tripleo-heat-templates/docker/services/neutron-ovs-dpdk-agent.yaml
```

## Kernel Args Configuration

In newton, the kernel args configuration and reboot of the node is achieved
using the `first-boot` scripts. In queens, the same has been achieved by using
the `host-config-and-reboot.yaml` environment file. During the upgrade, it is
required to maintain the same `first-boot` scripts. Adding the new environment
file will result in an additional reboot during the upgrade process. There is
on going work on supporting the new environment for FFU upgrade, until then it
should not be used.


## Parameters Changes

<span style='color:red;'>**_Note:_**</span> Below changes are only for
information. As the `first-boot` scripts will be still using the older
parameters, it is not required to change to new parameters during upgrade.
`first-boot` scripts will not be used during the upgrades, but after the
upgrade, if there is any scale-up operations, then it will be used. Until the
dependency of `first-boot`  is removed, this parameter change is not required.

OvS-DPDK parameters defined in newton are deprecated in queens (but it is
still supported). Those deprecated parameters has been removed the rocky
(OSP14) release. Below are the list of deprecated and their alternatives:

<table class="table"> <thead> <tr><td><b>Deprecated
Parameter</b></td><td><b>New Parameter</b></td></tr> </thead> <tbody>
<tr><td>HostCpusList</td><td>OvsDpdkCoreList</td></tr>
<tr><td>NeutronDpdkCoreList</td><td>OvsPmdCoreList</td></tr>
<tr><td>NeutronDpdkMemoryChannels</td><td>OvsDpdkMemoryChannels</td></tr>
<tr><td>NeutronDpdkSocketMemory</td><td>OvsDpdkSocketMemory</td></tr>
<tr><td>NeutronDpdkDriverType</td><td>OvsDpdkDriverType</td></tr> </tbody>
</table>

