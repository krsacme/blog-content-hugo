+++
date = "2017-09-11T17:57:16+05:30"
title = "TripleO Role-Specific Parameters"
tags = ["openstack","tripleo", "role"]
highlight = true
math = false

[header]
  caption = ""
  image = ""

+++

OpenStack installer TripleO provides a flexibility to the operators to define
their own custom roles. A custom role can be defined by associating a list of
predefined (or custom-defined) services. A TripleO service can be associated
with multiple roles, which brings in the requirement to keep the parameter to
be role-specific. This has been achevied in **Pike** release by introducing a
new parameter **RoleParameters** to the TripleO service template.
<!-- more-->

By default, not all parameters are role-specific. Additional implementation
has to be provided on a TripleO service template to enable role-specific
parameters support. With role-specific parameters supported, a parameter can
be provided as role-specific or global (as like existing). The preference is
given to the role-specific parameter if provided, else the global parameter
values are applied.

All the parameters which should be applied to a particular role should be
provided with `<RoleName>Parameters` as like below example. Here, `<RoleName>`
is the name of the role as defined in the `roles_data.yaml`.

Lets us consider we have multiple compute roles in a cluster like

* Compute
* ComputeOvsDpdk
* ComputeSriov

Below example demonstrates how to target a parameter specific to a role. Here
the parameter `NovaReserveHostMemory` will have the value as 1024 in
**Compute** and **ComputeSriov** roles, where as the role **ComputeOvsDpdk**
will have the value as 4096.

```yaml
parameter_defaults:
  NovaReservedHostMemory: 1024
  ComputeOvsDpdkParameters:
    NovaReservedHostMemory: 4096
```

_Facts about using role-specific:_

* It is possible provide only the role-specific parameter without global
  definition, in which case the default value of the service is applied for
  the other roles
* It is possible to define a parameter for all the roles as role-specific
  with/without global definition

### Role-specific parameters in Pike (OSP12)

##### Service: nova_compute
  * NovaVcpuPinSet
  * NovaReservedHostMemory

##### Service: neutron_ovs_dpdk_agent
  * OvsDpdkCoreList
  * OvsDpdkMemoryChannels
  * OvsDpdkSocketMemory
  * OvsPmdCoreList
  * NeutronDatapathType
  * NeutronVhostuserSocketDir

##### Service: neutron_sriov_agent
  * NeutronPhysicalDevMappings
  * NeutronExcludeDevices
  * NeutronSriovNumVFs

##### Service: opendaylight_ovs
  * HostAllowedNetworkTypes
  * OvsEnableDpdk
  * VhostuserSocketDir
  * OvsVhostuserMode
  * OpenDaylightProviderMappings

## Other Versions
The role-specific feature is available in Pike release (OSP12), where as the
custom roles is available from Newton release (OSP10). If it is required to
support role-specific parameter prior to Pike releases, it is possible with a
workaround for certain template parameters.

### Parameters used by Puppet
Instead of using the template parameter, the corresponding parameter's hiera
variable could be overriden for a particular role with `<RoleName>ExtraConfig`
parameter.

For the same example as explained above, it is possible to provide it via
hiera value. First we need to find the hiera value of the parameter
`NovaReservedHostMemory` form the [tripleo-heat-templates] repository, which
is `nova::compute::reserved_host_memory`. Then following sample overrides the
hiera value for the `ComputeOvsDpdk` role:

```yaml
parameter_defaults:
  NovaReservedHostMemory: 1024
  ComputeOvsDpdkExtraConfig:
    nova::compute::reserved_host_memory: 4096
```

**_Note:_**
Overriding via hiera varibles for complex datatypes like map or list should be
analyzed well in order to understand how the parameter is accepted in the
puppet.

### Parameters in first-boot scripts (cloud-init)
In OSP10, there has been few configurations which are done as part of the
[first-boot] scripts, like `ComputeKernelArgs`, `HostIsolatedCoreList`. If we
need to support role-specific parameters for these parameters, the
implementation of the first-boot script has to enhanced to handle multiple
parameters. Note that it is entierly in the deployer's purview to modify the
script as per their defined roles. The below sample code provides an overview
of how this could be acheived considering the above examples.

```yaml
heat_template_version: 2014-10-16
parameters:
  ComputeSriovKernelArgs:
    type: string
  ComputeSriovHostnameFormat:
    type: string
    default: computesriov
  ComputeOvsDpdkKernelArgs:
    type: string
  ComputeOvsDpdkHostnameFormat:
    type: string
    default: computeovsdpdk

resources:
  userdata:
    type: OS::Heat::MultipartMime
    properties:
      parts:
      - config: {get_resource: compute_kernel_args}

  compute_kernel_args:
    type: OS::Heat::SoftwareConfig
    properties:
      config:
        str_replace:
          template: |
            #!/bin/bash
            set -x
            SRIOV=$COMPUTE_SRIOV_HOSTNAME_FORMAT
            SRIOV=$(echo $SRIOV | sed  's/\%index\%//g' | sed 's/\%stackname\%//g') ;

            DPDK=$COMPUTE_DPDK_HOSTNAME_FORMAT
            DPDK=$(echo $DPDK | sed  's/\%index\%//g' | sed 's/\%stackname\%//g') ;

            if [[ $(hostname) == *$SRIOV* ]] ; then
              # Scripts for SR-IOV role
            fi

            if [[ $(hostname) == *$DPDK* ]] ; then
              # Scripts for OvS DPDK role
            fi
          params:
            $KERNEL_ARGS_SRIOV: {get_param: ComputeSriovKernelArgs}
            $COMPUTE_SRIOV_HOSTNAME_FORMAT: {get_param: ComputeSriovHostnameFormat}
            $KERNEL_ARGS_DPDK: {get_param: ComputeOvsDpdkKernelArgs}
            $COMPUTE_DPDK_HOSTNAME_FORMAT: {get_param: ComputeOvsDpdkHostnameFormat}

```



[tripleo-heat-templates]: https://github.com/openstack/tripleo-heat-templates
[first-boot]: https://github.com/krsacme/tht-dpdk/blob/master/osp10_ovs26/first-boot.yaml
