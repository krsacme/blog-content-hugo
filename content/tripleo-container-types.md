---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2017-09-13T14:57:16+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- openstack
- tripleo
- container
title: TripleO Container - Types
type: post
---

In **Pike** release, TripleO container deployment has been completely
redesigned, in a way that it is backward comptible with _baremetal_ deployment
and re-using most of the existing parts of TripleO. In this post, I would like
to detail the different stages of a container deployment and the associated
config files and log files.
<!-- more-->

With Pike release, most of the OpenStack services are containerized, leaving
some of the platform services like OpenvSwitch to be completed with subsequent
releases.

## Types of Container
As of Pike release all the container running in TripleO are based out of Kolla
image format. But we can differentiate the containers on the overcloud node
based on their purpose and how they are run.

### Service Containers
In case of a service, a container will be running all the time in _detached_
mode with restart policy as _always_. A typical definition of the a service
container is shown below, in which ```restart``` is set as ```always``` and
then ```detach``` will be left as default which is ```true```. The entry point
for this type of container will be ```kolla_start``` which will have the
command defined via kolla config file.

```yaml
outputs:
  role_data:
    description: Role data for the Libvirt service.
    value:
      docker_config:
        step_3:
          nova_virtlogd:
            start_order: 0
            image: {get_param: DockerNovaLibvirtImage}
            net: host
            pid: host
            privileged: true
            restart: always
```

### One-Off Containers
But in some cases where a bootstrap of a service is required, before running
the actual service or a post action need to be done after deploying the actual
service, a one-off container will be run. It will run only once in a
deployment and will exit after the deployment. The significant properties of
these types of container ```detach``` which should be set as ```false``` and
the ```restart``` property is set to default which is no restart. Instead of
the actual entry point of the container image, a custom ```command``` will be
provided along with this container definition. A sample definition is shown
below.

```yaml
outputs:
  role_data:
    description: Role data for the Libvirt service.
    value:
      docker_config:
        step_4:
          nova_libvirt_init_secret:
            image: {get_param: DockerNovaLibvirtImage}
            privileged: false
            detach: false
            command:
              - /bin/bash
              - -c
              ...
```
