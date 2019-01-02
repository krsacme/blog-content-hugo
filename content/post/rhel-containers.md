+++
type = "post"
date = "2019-01-02T09:00:00+05:30"
title = "Building RHEL Containers with Additional Repositories"
tags = ["rhel", "containers"]
highlight = true
math = false
author = "Saravanan KR"

[header]
  caption = ""
  image = ""

+++

When building RHEL containers on some scenarios may require to install a
specific package inside the container from a generic RHEL repository. In such
cases, the command `yum install` can be used inside the docker file to install
the package. The RHEL subscription of the host (on which the docker build is
executed), will be used inside the containers.

By default, the repository `rhel-7-server-rpms` will be enabled inside the
container. Here is a sample file to install `make` inside the the RHEL
container.

```
# Dockerfile
FROM rhel7

RUN yum install git

```

With the same approach,  installing `golang` will fail because, by default,
only the repository `rhel-7-server-rpms` is enabled inside the container. As
`golang` is provided from `rhel-7-server-optional-rpms` repository, following
changes required in the yum install command in the Docker file.

```
# Dockerfile
FROM rhel7

RUN yum install git golang --enablerepo="rhel-7-server-optional-rpms"

```
