+++
type = "post"
date = "2017-02-17T12:10:00"
tags = ["tripleo", "sriov"]
title = "SR-IOV Deployment steps for Tripleo (Newton RHOSP10)"
highlight = true
math = false

[header]
  caption = ""
  image = ""

[author]
  DisplayName = "Saravanan KR"
  Link = "krsacme"
+++

Red Hat OpenStack Platform v10 can be deployed via OSP-director by enabling
SR- IOV on the compute overcloud nodes. This post is going to detail the steps
involved in this deployment, along with other required details of SR-IOV
deployment.
<!--more-->

The following list of changes has to be added on top of the usual OSP10
deployment in order to enable SR-IOV in compute nodes. Here we are going
disucss on enabling SR-IOV on the Compute role itself. The steps for deploying
it as a new custom role along with existing Compute role will be added shortly
and will be linked here.

 * Environment file
 * Parameters

### Environment file
SR-IOV can be enabled by adding the ``neutron-sriov.yaml`` environment file to
the deploy command, like below

``` bash
  -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-sriov.yaml
```
This only enables the SR-IOV composable service in the ``Compute`` role.
Additionally, we need to give the following parameters to assist the
succesfully deployment.

### Parameters

Following are the list of parameters which needs to be configured addit

* [NeutronPhysicalDevMappings](#neutronphysicaldevmappings)
* [NeutronSriovNumVFs](#neutronsriovnumvfs)
* [NovaPCIPassthrough](#novapcipassthrough)
* [NovaSchedulerDefaultFilters](#novaschedulerdefaultfilters)
* [NovaSchedulerAvailableFilters](#novascheduleravailablefilters)
* [NeutronSupportedPCIVendorDevs](#neutronsupportedpcivendordevs)


##### NeutronPhysicalDevMappings

This parameter contains the list of physical network to the physical device
mapping.

Example:
``` yaml
parameter_defaults:
  NeutronPhysicalDevMappings: "datacentre:ens20f2"
```

##### NeutronSriovNumVFs

This parameter contains the number of VFs to be created on a physical
interface as a list. The number of VFs will be enabled on the corresponding
interface.

```bash
    echo "<N>" > /sys/class/net/<interface>/device/sriov_numvfs
```

As this configuration is transitent, there is a boot-up script written on the
SR-IOV deployed node via ``ifup-local`` to invoke this command every time the
overcloud node reboots.

Example:
``` yaml
parameter_defaults:
  NeutronSriovNumVFs: "ens20f2:4"
```
##### NovaPCIPassthrough
This parameter contains the white list of PCI devices available to VMs.

Examples:

This example is to illustrate the different format of this parameter that can
be given in the templates. Choose the best format which suits your deployment.

``` yaml
parameter_defaults:
    NovaPCIPassthrough:
      - devname: "ens20f2"
        physical_network: "datacentre"
      - address: "*:0a:00.*"
      - address: ":0a:00."
        physical_network: "datacentre"
      - vendor_id: 1137
        product_id: 0071
      - vendor_id: 1137
        product_id: 0071
        address: "0000:0a:00.1"
        physical_network: "datacentre"
```
Note::

  Following format is an invalid format, as they specifiy mutually
  exclusive options.
``` yaml
parameter_defaults:
    NovaPCIPassthrough:
      - devname: "ens20f2"
        physical_network: "datacentre"
        address: "*:0a:00.*"
```
All these details are specificed in the nova.conf documenation for
``pci_passthrough_whitelist``. This option is being moved from the ``DEFAULT``
secion to the ``pci`` section and renamed as ``passthrough_whitelist`` in the
Ocata release.

##### NovaSchedulerDefaultFilters

``` yaml
parameter_defaults:
  NovaSchedulerDefaultFilters: ['RetryFilter','AvailabilityZoneFilter','RamFilter','ComputeFilter','ComputeCapabilitiesFilter','ImagePropertiesFilter','ServerGroupAntiAffinityFilter','ServerGroupAffinityFilter','PciPassthroughFilter']
```

##### NovaSchedulerAvailableFilters

``` yaml
parameter_defaults:
  NovaSchedulerAvailableFilters: ["nova.scheduler.filters.all_filters","nova.scheduler.filters.pci_passthrough_filter.PciPassthroughFilter"]
```

##### NeutronSupportedPCIVendorDevs

**This parameter id deprecated in OSP10.** This parameter contains the list of
vendor id and product id of the **Virtual Function (VF)** which will be used
for SR-IOV deployment. Usually deployers make a mistake of assuming this as a
the vendor and product id of the Physical Function (PF) which is not the case.
As of now there is no straighforward way to identify these ids. Either we
should get this information from nic vendor or we have exeucute below command
in one of the node which used for deployment:

``` bash
   echo "1" > /sys/class/net/<interface>/device/sriov_numvfs
   lspci -nn  | grep -i 'network\|ether'
```

Example:
``` yaml
parameter_defaults:
  NeutronSupportedPCIVendorDevs: ['8086:154c','8086:10ca','8086:1520']
```

_Note:_

``NeutronMechanismDrivers`` parameter is modified in the SR-IOV TripleO
environment file to include ``sriovnicswitch``. On adding the environment file
``neutron-sriov.yaml`` to the deploy command, no need to override this
paramter in tempaltes.

### Complete List of SR-IOV Template Parameters

Create a file ``/home/stack/sriov-environment.yaml`` with below contents
modified to your environment:
``` yaml
parameter_defaults:
  NeutronSupportedPCIVendorDevs: ['8086:154c','8086:10ca','8086:1520']
  NeutronPhysicalDevMappings: "datacentre:ens20f2"
  NeutronSriovNumVFs: "ens20f2:4"
  NovaPCIPassthrough:
    - devname: "ens20f2"
      physical_network: "datacentre"
  NovaSchedulerDefaultFilters: ['RetryFilter','AvailabilityZoneFilter','RamFilter','ComputeFilter','ComputeCapabilitiesFilter','ImagePropertiesFilter','ServerGroupAntiAffinityFilter','ServerGroupAffinityFilter','PciPassthroughFilter']
  NovaSchedulerAvailableFilters: ["nova.scheduler.filters.all_filters","nova.scheduler.filters.pci_passthrough_filter.PciPassthroughFilter"]
```

Deploy Command:
``` bash
  openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/neutron-sriov.yaml \
    -e /home/stack/sriov-environment.yaml \
    ...
```