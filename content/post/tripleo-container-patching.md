+++
type = "post"
date = "2019-04-10T09:00:00+05:30"
title = "TripleO Containers Patching"
tags = ["tripleo", "container"]
highlight = true
math = false
author = "Saravanan KR"

[header]
  caption = ""
  image = ""

+++

With the support of only container based deployments, it is important to know
that how to edit files used for development to verify end to end deployment
with the changes. The puppet manifests changes has to be applied to
overcloud-full image, same as existing, in the folder
/usr/share/openstack-puppet/modules. And code changes specific to a container
has to be applied on the container.

## Updating a Package in a Container

* Download the RPMs to be update to on the container image, in this post, lets
  see how to override, openvswitch2.10-ovn-host RPM for the ovn-controller
  container image.

* Create a Dockerfile from the existing ovn-controller image

```
  FROM 192.168.10.1:8787/rhosp14/openstack-ovn-controller:2019-02-21.1
  ADD openvswitch2.10-ovn-host-2.10.0-54.el7fdn.x86_64.rpm \
    /openvswitch2.10-ovn-host-2.10.0-54.el7fdn.x86_64.rpm
  RUN rpm -Uvh /openvswitch2.10-ovn-host-2.10.0-54.el7fdn.x86_64.rpm
  ENTRYPOINT kolla_start
```

* Build the image with a custom tag "patched" and push it to the container registry

```bash
  $ docker build . -t 192.168.10.1:8787/rhosp14/openstack-ovn-controller:patched
  $ docker push 192.168.10.1:8787/rhosp14/openstack-ovn-controller:patched
```

* Updating the image after the deployment in the overcloud node

  * Update the image tag to "patched" in the container startup config scripts,
    for ovn-controller, update the file
    hashed-docker-container-startup-config-step_4.json with this new tag.

  * Apply the modified config file using paunch command

```bash
  $ paunch apply --config-id tripleo_step4 \
      --file /var/lib/tripleo-config/hashed-docker-container-startup-config-step_4.json
```

* Using the updated image as part of the deployment
  * Create an environment file to override the image for ovn-controller

```yaml
  parameter_defaults:
    DockerOvnControllerImage: 192.168.10.1:8787/rhosp14/openstack-ovn-controller:patched
```

  * Add this environment file to the deploy command

## Modify Source Code in Container

Following steps allows to modify a container image in undercloud and upload it
to registry so that it could be applied to all the nodes during the
deployment.

* Run the run command to enter to the container with bash prompt, specify a
  custom name for this instance

```bash
  $ docker run --name server-ovn-patch -it -u root \
      192.168.10.1:8787/rhosp14/openstack-neutron-server-ovn:2019-02-21.1 bash
```

* Modify the required files and exit the container

* Commit the run instance to a image with specific tag and push it to the
  registry

``` bash
  $ docker commit --change "ENTRYPOINT kolla_start" server-ovn-patch \
      192.168.10.1:8787/rhosp14/openstack-neutron-server-ovn:patched
  $ docker push 192.168.10.1:8787/rhosp14/openstack-neutron-server-ovn:patched
```
* Using the updated image as part of the deployment
  * Create an environment file to override the image

```yaml
  parameter_defaults:
    DockerNeutronApiImage: 192.168.10.1:8787/rhosp14/openstack-neutron-server-ovn:patched
```

  * Add this environment file to the deploy command
