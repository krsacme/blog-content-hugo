+++
date = "2017-03-12T11:09:14+05:30"
title = "TripleO Heat Templates - Basics"
tags = ["tripleo", "heat", "templates"]
highlight = true
math = false
summary = """
Basic explanation of the TripleO Heat Templates and its structure.
"""
[header]
  image = ""
  caption = ""

+++

TripleO deployment is primarily based on Heat orchestration to co-ordinates
the creation and deployment of different resources an OpenStack cluster
deployment. TripleO is a feature rich installer, which is highly customizable
for specific requirements and hardware dependency. This post is going to
explain the usage of Heat templates in TripleO installer.

## Heat Orchestration Templates (HOT) Format

The format of the HOT template is:

```yaml
heat_template_version: ocata

description:
  # a description of the template

parameter_groups:
  # a declaration of input parameter groups and order

parameters:
  # declaration of input parameters

resources:
  # declaration of template resources

outputs:
  # declaration of output parameters

conditions:
  # declaration of conditions
```

The explanation of each section and its functionalities can be found in the [HOT
Spec][1].

### Types of Resources
  * Heat Pre-defined Resource Type - Example: ```OS::Heat::SoftwareConfig```

  * OpenStack specific Pre-defined Resource Type - Example:
    ```OS::Nova::Server```

  * Custom Resource Type - Example: ```OS::TripleO::Network::External```. It
    will reference to another heat template file itself, a kind of nested
    templating


## Jinja Templating
A important concept to know before going in deep of TripleO templates, is the
usage of Jinja templating in the heat templates. With the introduction of
composable roles in newton cycle, which is an architectural change of bringing
feature association with any node instead of static association, it brings in
the requirement of defining the heat resources dynamically for a deployment.
For this dynamic heat resource definition, Jinja templating engine is used to
define the heat resources based on the role definition. More about the
detailed explanation of composable roles and its entities will be explained in
following posts.


## TripleO Heat Template - Files to Know

### overcloud.j2.yaml
Top-level heat template file for TripleO deployment. This file contains the
```overcloud``` stack definition template.

### overcloud-resource-registry-puppet.j2.yaml
Custom resource type definition of TripleO heat templates are listed in this
file. Note, that deployer can define their own custom type which can be
associated with any file or pre-define resource and provide as input to the
deployment.

### roles_data.yaml
A file which contains the roles and its definition. This file need to be
customized in order to define the customized roles for a deployment. Each role
is associated with a set of features (services), which are pre-defined.

### ```environments``` directory
Adding a feature in TripleO deployment involves defining the custom heat
resource type and the required parameters for the features. An environment file
is provided for each feature available with TripleO installer in this
``environments`` directory. Deployer can use this environment file during the
plan creation (stack deployment).


## Understanding Nested Templates in TripleO
TripleO initiates the heat stack deployment with a top level template
```overcloud.j2.yaml``` which is of Jinja template. Before the heat stack
creation, all the ```.j2.yaml``` files in the templates folder will be applied
to Jinja templating with the input from the ```roles_data.yaml``` file (which
contains the role definition).

Then the top-level stack creation is initiated which will build all the
resources of this stack. Based on the resources types, the corresponding
action will be taken by the heat engine. In case of custom resource types, the
associated templates file to the resource type will be created as a nested
stack to the resource and the parent stack which has this resource.

### Sample Stack Output Tree
Lets take an example to understand the nested stacks.

```yaml
overcloud:
  resources:
    Networks:
      type: OS::TripleO::Network
      stacks:
        overcloud-Networks-f53ijhkio762:
          resources:
            ExternalNetwork:
              type: OS::TripleO::Network::External
              stacks:
                overcloud-Networks-f53ijhkio762-ExternalNetwork-h5w3sdi4wzbm:
                  resources:
                    ExternalNetwork:
                      type: OS::Neutron::Net
                    ExternalSubnet:
                      type: OS::Neutron::Subnet
```

The top level stack is named as ```overcloud``` which is represened by the
file ```overcloud.j2.yaml```, which contains lots of resources under its
definitiong. Lets take one of the resource, ```Networks```, whic is defined as
custom heat resource type ```OS::TripleO::Networks```. This custom resource
type is mapped to a heat template file ```network/networks.yaml``` in the
resource registry file. This will create a nested stack under the resource
```Networks``` with stack name as ```overcloud-Networks-xxxxxx```.

This nested stack, which is in the second level, contains multiple resource
defined in the template file, as ExternalNetwork, TenantNetwork,
InternalNetwork, etc,. Consider the resource ```ExternaleNetwork```, which is
also a custom heat resource type, which is defined in the registry file as
```OS::TripleO::Network::External```. This resource type can be be mapped to
different template files based on the type of the deployment based on IPV4 or
IPv6. For IPv4 deployment, the template file ```environments/network-
isolation.yaml``` provides the mapping to this custom type of the template
file ```network/external.yaml```. This template file will create a nested
stack under the resource ```ExternalNetwork``` with stack name as
```overcloud-Networks-xxxxxx-ExternalNetwork-yyyyyy``` .

This nested stack, which is in the third level, also contains two resource
defined in the template file as ExternelNetwork and ExternalSubnet. Consider
the resource ```ExternalNetwork```, which is a heat defined resource type
```OS::Neutron::Net``` (provided by OpenStack resource definitions). This
resource definition is embedded with in the heat engine.



### Useful Commands

  * ```openstack stack resource list overcloud``` provides the list of
    resources defined in the top level stack ```overcloud```

  * ```openstack stack resource show overcloud Networks``` provides the
    details of the resource ```Networks``` from the top level stack. The
    attribute ```links``` in the detailed view of the resource provides the
    resource's and its associated stack heat engine HTTP links. This link is a
    JSON list with an Object containing ```href``` (heat engine HTTP link) and
    ```rel``` (type of the link). The list of possible types of links are:
    * ```self``` - Provides the HTTP link of this resource
    * ```stack``` - Provides the HTTP link of the stack to which this resource
      belongs too
    * ```nested``` - Provides the HTTP link of the nested stack of this
      resource. The name of the nested stack can be found from this link
      itself. With this link, in the sample show below, the nested stack name
      can be identified as ```overcloud-Networks-f53ijhkio762```.

```json
[
  {
    "href": "http://172.18.0.1:8004/v1/8094ede366ef4c6f8dd7d8e6876fa910/stacks/overcloud/ea2098fb-f41e-400c-ad19-1ed8f83476b3/resources/Networks",
    "rel": "self"
  },
  {
    "href": "http://172.18.0.1:8004/v1/8094ede366ef4c6f8dd7d8e6876fa910/stacks/overcloud/ea2098fb-f41e-400c-ad19-1ed8f83476b3",
    "rel": "stack"
  },
  {
    "href": "http://172.18.0.1:8004/v1/8094ede366ef4c6f8dd7d8e6876fa910/stacks/overcloud-Networks-f53ijhkio762/aadaf2af-a1b9-412a-a152-1e2ac0488a1c",
    "rel": "nested"
  }
]
```

  * ```openstack stack resource list overcloud-Networks-f53ijhkio762```
    provides the list of resource's associated with this nested stack

  * ```openstack stack list --nested``` provides the list of all stacks
    including the nested stacks

  * ```openstack stack resource list overcloud -n2``` provides the list of all
    the resources associated with top-level and the nested stack with 2 level.


[1]: https://docs.openstack.org/developer/heat/template_guide/hot_spec.html#template-structure
