+++
Categories = ["TripleO","DPDK"]
date = "2016-12-20T17:05:52+05:30"
title = "DPDK - Assigning CPUs for Poll Mode Driver (PMD)"
Tags = ["TripleO","DPDK"]
#Description = "Blog details the steps involved in deciding the DPDK PMD CPUs"
menu = "main"

+++
OVS-DPDK is a critical requirement in the NFV world with OpenStack. TripleO
has integrated the deployment of OVS-DPDK with the newton release, till then
it used to be a manual step post deployment. There are a list of parameter
which has to be provided for the deployment with OVS-DPDK. One such parameter
is *NeutronDpdkCoreList*. This blog details the factors to be considered in
order to decide the value for this parameter.

### CPU List for PMD
*NeutronDpdkCoreList* parameter configures the list of CPUs to be used by the
Poll Mode Drivers. Following criteria should be considered to deliver an
optimal performance with OVS-DPDK:

  * CPUs should be associated with NUMA node of the DPDK interface
  * CPU siblings (in case of Hyper-threading) should be added to the together
  * CPU 0 should always be left for the host process
  * CPUs assigned to PMD should be isolated so that the host process does
    not use those
  * CPUs assigned to PMD should be exclued from nova scheduling using
    *NovaVcpuPinset*

#### Fetching the Information
  * ```lscpu``` command will give list of CPUs associated with the NUMA node
  * ```cat /sys/devices/system/cpu/<cpu>/topology/thread_siblings_list```
    command will give the siblings of a CPU
  * ```cat /sys/class/net/<interface>/device/numa_node``` command will
    give the NUMA node associated with an Interface

#### Sample Configuration

![NUMA Node Mapping](/numa_mapping_for_dpdk_in_tripleo.jpg)

For better explanation, let us consider a baremetal machine with below specification.
  * A machine with 2 NUMA nodes
  * Each NUMA node has 4 cores and a 1 NIC
  * Hyper-threading value as 2, so each core is running 2 threads (siblings)
  * Each NUMA node eventually contains 8 CPUs

To configure NIC1 as DPDK, following should be configuration
```
NeutronDpdkCoreList: "'2,3'"
NeutronDpdkSocketMemory: "'1024'"
```

To configure NIC2 as DPDK, following should be configuration
```
NeutronDpdkCoreList: "'8,9'"
NeutronDpdkSocketMemory: "'0,1024'"
```