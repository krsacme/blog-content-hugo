---
author:
  DisplayName: Saravanan KR
  Link: krsacme
date: "2020-09-18T09:00:00+05:30"
header:
  caption: ""
  image: ""
highlight: true
math: false
tags:
- kubernetes
- operators
- catalog
title: Kubernetes Operators - Custom Catalog
type: post
---

There are multiple resources involved in a operator `Subscription` in a
OpenShift 4 cluster. This document explains different blocks of subscribing an
operator. Also explains how to include a custom catalog for development or
expriement purposes. The resources used in this are based on OpenShift 4.5 and
Operator SDK v1.0 versions. This blog assumes that the reader has fair
knowledge of operators.

Resources involved in creating and subscribing an custom operator are:

 * Operator Bundles
 * CatalogSource
 * OperatorGroup
 * Subscription
 * ClusterServiceVersion

![CatalogSource Contents](/img/operator/operator-pkg1.png)

This diagram depicts how the operator image (`testpmd-operator`) and its
metadata information is encompassed in to a operator bundle image
(`testpmd-operator-bundle`). Multiple operator bundle images can be sourced
from a single  catalog index image (`nfv-example-cnf-catalog`). Using this
image, a `CatalogSource` resource object should be created to source the
`testpmd-operator`.

## Operator Bundles

Operator SDK v1.0 by default scaffolds the necessary file structure and make
commands to generate the operator bundle images. Operator bundle image
consists of the CRDs, CSVs and RBACK definitions of an operator specific to a
version.

When the operator is scaffolded for operator sdk v1.0 version, it creates the
Makefile with commands to generate the bunlde files and bundle container
image. Executing the make commands `make bundle` and `make bundle-build`
should create the bundle container image for the operator, which need to be
pushed to a registry.

## CatalogSource

`CatalogSource` is the source for operators and its different versions. When a
`CatalogSource` is created in a cluster, it creates a database consisting of
all the operators that this catalog can source. The operator will be exposed
as a `PackageManifests` to know what the operator consists of and its
different versions.

The catalog source container image is created using the `opm index add`
command, by providing operator bundle images. A sample command to create a
catalog index image using the opm tool is:

```bash
opm index add -u podman --bundles \
  quay.io/krsacme/testpmd-operator-bundle:v0.1.1,quay.io/krsacme/trex-operator-bundle:v0.1.1 \
  --tag quay.io/krsacme/nfv-example-cnf-catalog:v0.1.1
```

Once the operator catalog index image is created, then create a
`CatalogSource` resource to provide the custom operators in the cluster:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: nfv-example-cnf-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/krsacme/nfv-example-cnf-catalog:v0.1.1
  displayName: NFV Example CNF Catalog
  updateStrategy:
    registryPoll:
      interval: 30m
```

## OperatorGroup

An `OperatorGroup` resource provides the list of namespaces or a selector for
namespace on which the RBAC should be generated for its member operators. The
target namespace will be applied to all the CSVs which will be installed in
this namespace.

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: example-cnf-operator-group
  namespace: example-cnf
spec:
  targetNamespaces:
    - example-cnf
 ```

## Subscription

A `Subscription` resource initiates an operator installation from the
specified catalog source with the specified channel. Once a `Subscription` is
created, then OLM creates a job to fetch the required CRDs, CSVs and RBACs for
the specified operator from the catalog source (via the operator bundle image)
and installs the CSV to the required namespace.

![CatalogSource Contents](/img/operator/operator-pkg2.png)

Create a `Subscription` for the `testpmd` operator from the custom catalog,
like:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: testpmd-operator-subscription
  namespace: example-cnf
spec:
  channel: "alpha"
  name: testpmd-operator
  source: nfv-example-cnf-catalog
  sourceNamespace: openshift-marketplace
```

On creating a subscription, OLM creates a job to process the request and
install the required resoures. A snaphot of the job and pods associated with
this operation is shown below:

```
$ oc -n openshift-marketplace get all

NAME                                                                        COMPLETIONS   DURATION   AGE
job.batch/37eae4d05fc938390e8ad0264dd3b703722ad98c1a9f565b585b14d6eed6425   1/1           19s        24h
job.batch/8d971dd002fd5e41e1e17949ff724b5bf52e8cadfe25f09b0887b5f67e24523   1/1           19s        24h

NAME                                                                  READY   STATUS      RESTARTS   AGE
pod/37eae4d05fc938390e8ad0264dd3b703722ad98c1a9f565b585b14d6eef4r6z   0/1     Completed   0          24h
pod/8d971dd002fd5e41e1e17949ff724b5bf52e8cadfe25f09b0887b5f67e2kjv2   0/1     Completed   0          24h
pod/nfv-example-cnf-catalog-z969l                                     1/1     Running     0          14m
```

The job that is created above gets the CRD and CSV from the operator bundle
and then installs both in the cluster. A snapshot of how the CSVs will be
installed:

```bash
$ oc get crd  | grep 'testpmds\|trex\|NAME'
NAME                                                        CREATED AT
testpmds.examplecnf.openshift.io                            2020-09-17T11:40:41Z
trexconfigs.examplecnf.openshift.io                         2020-09-17T11:40:44Z

$ oc -n example-cnf get csv
NAME                      DISPLAY            VERSION   REPLACES   PHASE
testpmd-operator.v0.1.1   TestPMD Operator   0.1.1                Succeeded
trex-operator.v0.1.1      TRex Operator      0.1.1                Succeeded
```

The operator deployment will be done by the CSV installation, which will
create the deployment resource for the operator associated.

## ClusterServiceVersion

The `ClusterServiceVersion` resources contains both technical information of
the operator and the user interface info like logo, description. The CSV
contains the operator deployment resources and the RBAC rules that the
operator requires. As part of CSV installtion (which is triggered by the
creation of a `Subscription`), the operator will be deployed.

```yaml
$ oc -n example-cnf get all
NAME                                                       READY   STATUS    RESTARTS   AGE
pod/testpmd-operator-controller-manager-79fc6f64cb-c6vw8   1/1     Running   0          19h
pod/trex-operator-controller-manager-54db9f5b6d-n62x5      1/1     Running   0          19h

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/testpmd-operator-controller-manager   1/1     1            1           19h
deployment.apps/trex-operator-controller-manager      1/1     1            1           19h

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/testpmd-operator-controller-manager-79fc6f64cb   1         1         1       19h
replicaset.apps/trex-operator-controller-manager-54db9f5b6d      1         1         1       19h
```
