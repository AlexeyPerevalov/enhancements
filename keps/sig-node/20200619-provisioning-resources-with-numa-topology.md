---
title: Provisioning the Node Resources With NUMA Topology Information
authors:
  - "@AlexeyPerevalov"
  - "@swatisehgal"
owning-sig: sig-node
participating-sigs:
  - sig-node
  - sig-scheduling
reviewers:
  - "@dchen1107"
  - "@derekwaynecarr"
approvers:
  - "@dchen1107"
  - "@derekwaynecarr"
creation-date: 2020-06-19
last-updated: 2020-06-19
status: implementable
see-also:
  - "/keps/sig-scheduling/20200612-deducted-topology-manager.md"
---

# Exposure Node Resources With NUMA Topology Information

## Table of Contents

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Design Details](#design-details)
  - [Design based on CRI](#cri)
  - [Design based on podresources interface of the kubelet](#podresources)
  - [API](#api)
  - [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
  - [Annotation approach] (#annotation-approach)
  - [NUMA specification in ResourceName] (#numa-in-resourcename)
<!-- /toc -->

## Summary

Cluster which contains several nodes with NUMA topology and
enabled TopologyManager feature gate is not rare now. In such cluster
could be a situation when TopologyManager's admit handler on the kubelet
side refuses pod launching since pod's resources request doesn't sutisfy
selected TopologyManager policy, in this case pod should be rescheduled.
And so it can get stuck in the rescheduling cycle.
To avoid such kind of problem scheduler should choose a node considering topology
of the node and TopologyManager policy on the worker node.

## Motivation

For the sake of scheduling which is topology aware, resources with topology
information should be provisioned.
This KEP describes how it would be implemented.

### Goals

Provisioning resources with topology information.

### Non-Goals

 - modification of any public API
 - improving and as a result modification of the TopologyManager and its policies

## Proposal

Introduce standalone daemon which will be run on each node in the cluster as daemonset. It should
collect resources for running pods and provide it with CRD.

## Design Details

The design consists of part which describes how datum collected
and how it was provided.

Resources used by the pod could be obtained by podresources interface
of the kubelet and by CRI from container runtime.

To calculate available resources need to know all resources
which could be used by kubernetes. It could be calculated by
substracting resources of kube cgroup and system cgroup from
system allocatable resources.

### Design based on CRI

The containerStatusResponse returned as a response to the ContainerStatus rpc contains `Info` field which is used by the container runtime for capturing ContainerInfo.
```go
message ContainerStatusResponse {
      ContainerStatus status = 1;
      map<string, string> info = 2;
}
```

Containerd has been used as the container runtime in the initial investigation. The internal container object info
[here](https://github.com/containerd/cri/blob/master/pkg/server/container_status.go#L130)

The Daemon set is responsible for the following:

- Parsing the info field to obtain container resource information
- Identifying NUMA nodes of the allocated resources
- Identifying total number of resources allocated on a NUMA node basis
- Detecting Node resource capacity on a NUMA node basis
- Updating the CRD instance per node indicating available resources on NUMA nodes, which is referred to the scheduler


#### Drawbacks

The content of the `info` field is free form, unregulated by the API contract. So, CRI-compliant container runtime engines are not required to add any configuration-specific information, like for example cpu allocation, here. In case of containerd container runtime, the Linux Container Configuration is added in the `info` map depending on the verbosity setting of the container runtime engine.

There is currently work going on in the community as part of the the Vertical Pod Autoscaling feature to update the ContainerStatus field to report back containerResources
[KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20191025-kubelet-container-resources-cri-api-changes.md).


### Design based on podresources interface of the kubelet

Podresources interface of kubelet described in

pkg/kubelet/apis/podresources/v1alpha1/api.proto

it available for every process on the worker node by
unix domain socket situated by the following path:

filepath.Join(kl.getRootDir(), config.DefaultKubeletPodResourcesDirName)

it could be used to collect used resources on the worker node and to evaluate
its NUMA assignment (by device id)

For example
TODO add path on the real system

Currently it doesn't provide information from CPUManager only from
DeviceManager as well as information about memory which was used.
But information provided by podresource could be easily extended
without modification of API.

### API

Available resources with topology of the node should be stored in CRD. Format of the topology described
[in this document](https://docs.google.com/document/d/12kj3fK8boNuPNqob6F_pPU9ZTaNEnPGaXEooW1Cilwg/edit).


```go
// NodeResourceTopology is a specification for a Foo resource
type NodeResourceTopology struct {
       metav1.TypeMeta   `json:",inline"`
       metav1.ObjectMeta `json:"metadata,omitempty"`

       TopologyPolicy string
       Nodes   []NUMANodeResource   `json:"nodes"`
}

// NUMANodeResource is the spec for a NodeResourceTopology resource
type NUMANodeResource struct {
       NUMAID int
       Resources v1.ResourceList
}
```

The code for working with it is generated by https://github.com/kubernetes/code-generator.git
One CRD instance contains information of available resources of the appropriate worker node.

### Graduation Criteria

* The feature has been stable and reliable in the past several releases.
* Documentation should exist for the feature.
* Test coverage of the feature is acceptable.


## Implementation History

- 2020-06-22: Initial KEP published.

## Alternatives

The provisioning of the resourcees could be implemented alsy by another way.
Daemon can keep resources in node annotation or in the pod's annotation.
Also kubelet can provide additional resources with NUMA information in ResourceName.

### Annotation approach

Annotation of the node or pod it's yet another place for arbitrary information.

This approach doesn't have known side effects.


### NUMA specification in ResourceName

The representation of resource consists of two parts subdomain/resourceName. Where
subdomain could be omitted. Subdomain contains vendor name. It doesn't suit well for
reflecting NUMA information of the node as well as / delimeter since subdomain is optional.
So new delimiter should be introduced to separate it from subdomain/resourceName.

It might look like:
numa%d///subdomain/resourceName

%d - number of NUMA node
/// - delimeter
numa%d/// - could be omitted

This approach may have side effects.
