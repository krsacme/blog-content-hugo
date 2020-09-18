---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2017-10-16T14:57:16+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- openstack
- tripleo
- container
title: TripleO Container - Template Configs (Pike)
type: post
---

In this post, I would like to provide the details of the different types of
template config sections present in a typical docker service template file.
<!--more-->

There are few configurations which are present in the `puppet/service`
templates like `service_name`, which still have the same interpretation in the
container services in `docker/services` templates too. Apart from that, there
are few container specific configurations, which are being explained in below
sections:


### puppet_config
Specifies the puppet class `step-config` and the puppet resources `puppet-
tags` to be applied while enabling a service. By default, all the file
operation related puppet resources like file, concat, file_line, augeas, cron
are applied. In case a puppet class has a custom resource to be applied for a
container, it could be added via `puppet-tags`. Note that, these puppet tags
will be applied on a temporary container. If it is required to a run puppet to
configure host and network, it has to be run as one-off container as part of
the docker_config.

This is an intermediary step to utilize the existing puppet modules to deploy
containers. This is acheived by running a temporary container and invoking
the puppet in `puppet_step_config.pp` in the overcloud node, which is handled
by docker- puppet files in `/var/lib/docker-puppet` directory. The script
[docker- puppet.py][docker-puppet] is responsible to run this temporary
container and generate the required configuration files for a service in the
overcloud node.

When the `docker-puppet.py` runs for a service, it creates all the config
files, as an output of applying puppet resources, in the `/var/lib/config-data
/puppet- generated/` directory. For example, a `nova.conf` and other required
files will be created for `nova_compute` service. Some optimizations could be
applied based on services to generate once and use it multiple containers. For
example, `nova_libvirt` and `nova_compute` have the same set of config files
so both the services will have identical `puppet_config` to create
`nova_libvirt` config volume and mounted on respective containers. All these
generated config files will be mounted to the respective container.


### kolla_config
Kolla container is started based on the kolla config files `config.json`,
which provides the actual `command` to run inside the kolla container, along
with other configurations and permissions. All the kolla containers will have
the base scripts to read the config files and configure the container
accordingly. By default, kolla container looks for the file
`/var/lib/kolla/config_files/config.json` to be present if no env is provided,
refer [set_configs.py][reading-config-file] for understand how the config
files are read.

This section in a service file provide the content to be writtent to the kolla
config file of a container. For example, this section will generate a file
like `/var/lib/kolla/config_files/nova_libvirt.json` which will be written on
the overcloud node, and it will be mounted on to the `nova_libvirt` container
as `/var/lib/kolla/config_files/config.json`.


### docker_config
This section provides the list of containers to be associated with a service.
As explained in previous blog, the section can have two types of containers:
service containers and one-off containers. Container defintions are associated
with step so that the spawning of containers are ordered, similar to heat and
puppet's step in baremetal deployment. For example, in a compute role,
nova_libvirt container is started at step3 and nova_compute container is
started at step4. Additional to steps, each container could be associated
with a `start_order` property to define the order of spwaning within a step.

All the configs for a role will be accumlated as per the step definitions, and
written to the overcloud node at `/var/lib/tripleo-config/` directory. And
then, the `paunch` tool is used to spawn the containers, by reading the
configs from this directory.

### host_prep_tasks
This sections provides the list of ansible tasks which need to be applied on
the host to prepare the host for running a service. For example, logs
directory is created during the host preparation, which will be mounted on to
the container.


[docker-puppet]: https://github.com/openstack/tripleo-heat-templates/blob/master/docker/docker-puppet.py
[reading-config-file]: https://github.com/openstack/kolla/blob/5.0.0/docker/base/set_configs.py#L278
