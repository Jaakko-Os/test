# MetalKube Documentation

The MetalKube project exists to provide components that allow you to do bare
metal host management for Kubernetes.  MetalKube works as a Kubernetes
application, meaning it runs on Kubernetes and is managed through Kubernetes
interfaces.

[Operators](https://github.com/operator-framework/operator-sdk) are a key piece
of the MetalKube architecture as the method used to manage kubernetes
applications.

## MetalKube Component Overview

### Machine API Integration

Another set of components is being designed and built to provide integration
with the Kubernetes [Machine
API](https://github.com/kubernetes-sigs/cluster-api).

This first diagram represents the high level architecture:

![High Level Architecture](images/high-level-arch.png)

#### Machine API Actuator

The first component is the [Bare Metal
Actuator](https://github.com/metalkube/cluster-api-provider-bare-metal).  This
is the component with logic specific to this architecture for handling changes
to the lifecycle of Machine objects.  This actuator may be integrated with the
existing [Machine API
Operator](https://github.com/openshift/machine-api-operator).

#### Bare Metal Operator

The architecture also includes a new [Bare Metal
Operator](https://github.com/metalkube/bare-metal-operator), which includes the
following:

* A Controller for a new Custom Resource, BareMetalHost.  This custom resource
  represents an inventory of known (configured or automatically discovered)
  bare metal hosts.  When a Machine is created the Bare Metal Actuator will
  claim one of these hosts to be provisioned as a new Kubernetes node.
* In response to BareMetalHost updates, will perform bare metal host
  provisioning actions as necessary to reach the desired state.  It will do so
  by managing and driving a set of underlying bare metal provisioning
  components.
* The implementation will focus on using Ironic as its first implementation of
  the Bare Metal Management Pods, but aims to keep this as an implementation
  detail under the hood such that alternatives could be added in the future if
  the need arises.

## APIs

1. Enroll nodes by creating BareMetalHost resources.  This would either be
   manually or done by a component doing node discovery and introspection.

```
apiVersion: metalkube.org/v1alpha1
kind: BareMetalHost
metadata:
    name: baremetal-node-1
    labels:
        node-profile: master
spec:
    mac: 00:11:22:33:44:01
    image: http://www.example.com/images/image.qcow2
    image_sha256sum: 4ed79aace680d9af89bdcf8a042238031eb147b5c8825499aa3314269647c300
    management_interface:
        type: ipmi
        username: admin
        password: <SECRET>
        address: 192.168.45.101


apiVersion: metalkube.org/v1alpha1
kind: BareMetalHost
metadata:
    name: baremetal-node-2
    labels:
        node-profile: node
spec:
    mac: 00:11:22:33:44:02
    image: http://www.example.com/images/image.qcow2
    image_sha256sum: 4ed79aace680d9af89bdcf8a042238031eb147b5c8825499aa3314269647c300
    management_interface:
        type: idrac
        username: admin
        password: <SECRET>
        url: http://host.domain
```

2. Use the machine API to allocate a machine.

```
apiVersion: "cluster.k8s.io/v1alpha1"
kind: Machine
metadata:
    generateName: baremetal-master-
    labels:
        set: master
spec:
    providerConfig:
        value:
            apiVersion: "baremetalproviderconfig/v1alpha1"
            kind: "BareMetalProviderConfig"
            selector:
                node-profile: master


apiVersion: "cluster.k8s.io/v1alpha1"
kind: Machine
metadata:
    generateName: baremetal-node-
    labels:
          set: node
spec:
    providerConfig:
        value:
            apiVersion: "baremetalproviderconfig/v1alpha1"
            kind: "BareMetalProviderConfig"
            selector:
                node-profile: node
```

3. Machine is associated with an available BareMetalHost, which triggers
   provisioning of that host to join the cluster.  (Exact mechanism for this
   association is TBD).
