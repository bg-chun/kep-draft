---
title: Memory Manager
authors:
  - "@bg-chun"
  - "@ohsewon"
  - "@admanV"
owning-sig: sig-node
participating-sigs:
  - sig-node
reviewers:
  - TBD
  - "@alicedoe"
approvers:
  - TBD
  - "@oscardoe"
editor: TBD
creation-date: 2019-07-18
last-updated: 2019-07-18
status: implementable
see-also:
replaces:
superseded-by:
---

# Memory Manager

_Authors:_

- @bg-chun - Byonggon Chun &lt;bg.chun@samsung.com&gt;
- @ohsewon - Sewon Oh &lt;sewon.oh@samsung.com&gt;
- @admanV - Hyunsung Cho &lt;hs2.cho@samsung.com &gt;

## Table of Contents

- [Overview](#overview)
- [Motivation](#motivation)
  - [Related Features](#Related-features)
  - [Related issues](#Related-issues)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
- [Proposal](#proposal)
  - [Proposed Changes](#proposed-changes)
    - [New Component: Memory Manager](#new-component-memory-manager)
      - [Calculate NUMA node affinity for memory and hugepages](#)
      - [Provide topology hint for Topology Manager](#)
      - [Isolate memory and hugepages for a container](#)
      - [New Interfaces](#new-interfaces)
    - [Topology Manager changes](#topology-manager-changes)
    - [Internal Container Lifecycle changes](#internal-container-lifecycle-changes)
    - [Feature Gate and Kubelet Flags](#feature-gate-and-kubelet-flags)
- [Graduation Criteria](#graduation-criteria)
  - [Phase 1: Alpha (target v1.1x)](#phase-1-Alpha-target-v11x)
  - [Phase 2: Beta](#phase-1-Beta)
  - [GA (stable)](#ga-stable)

# Overview

NUMA Awareness is a well-known solution to boost performance for diverse use cases including DPDK and database. For those use cases, Kubernetes support NUMA-related features for containers such as CPU Manager, Device Manager, and Topology Manager which treat CPU, I/O devices via PCI-e, and NUMA topology, respectively; However, there's no feature for memory and hugepages. It can cause inter-NUMA node communication in memory access, which leads to increase an I/O latency and then decrease a performance of entire system.

To resolve the problem, Memory Manager is proposed for a solution to deploy pods and containers with guaranteed QoS class of memory and hugepages under NUMA awareness.

# Motivation

## Related Features

- [Node Topology Manager][node-topology-manager] is a feature that collects topology hints from various hint providers to calculate socket affinity for a container. Topology Manager judges if container can admit under pre-configured NUMA policy and socket affinity.

- [CPU Manager][cpu-manager] is a feature that provides a solution for CPU Pinning based on cgroups cpuset subsystem. It also offer CPU-related topology hint to Topology Manager.

- [Device Manager][device-manager] is a feature that provides topology hint of I/O devices via PCI-e to Topology Manager. The main objectivity of Device Manager is to allow I/O device's vendors to adverise their resources to kubelet to be used for container or pod.

- [Hugepages][hugepages] is a feature to allow container to consume pre-allocated hugepages with pod isolation of hugepages.

- [Node Allocatable Feature][node-allocatable-feature] is a feature to help to reserve computing resources to prevent resource starvation. The kube-reserved and system-reserved is used to reserve resources for kubelet and system(OS, etc). Now, the following resources can be reserved: cpu, memory, ephemral storage.

## Related issues

- [Hardware topology awareness at node level (including NUMA)][numa-issue]
- [Support Container Isolation of Hugepages][hugepage-issue]

## Goals

- Guarantee alignment of resources for isolating container's memory and hugepages along with CPU and I/O devices.

- Provide topology hint for memory and hugepages for Topology manager.

## Non-Goals
- Updating scheduler and pod spec is out of scope at this point.

- This proposal only focus on Linux based system.

## User Stories

### Story 1 : Networking Acceleration using DPDK

- DPDK(Data Plane Development Kit) is set of libraries to accelerate packet processing on userspace. DPDK requires dedicated resources and alignment of resources under NUMA awareness. There should be a way to reserve memory and hugepages with other computing resources on specific socket for containerized VNF using DPDK.

### Story 2 : Database

- Databases(e.g., Oracle, PostgreSQL, MySQL, MongoDB, SAP) consume massive memory and hugepages in order to reduce memory access latency and improve performance dramatically. For this improvement, resources (i.e. CPU, memory, hugepages, I/O devices) should be aligned.

# Proposal

## Proposed Changes

### New Component: Memory Manager

A new component of Kubelet enables NUMA-awareness for memory and hugepages. The main roles of this component are listed below.


#### Calculate NUMA node affinity for memory and hugepages
>NUMA node affinity represents that which NUMA node has enough capacity of memory and/or hugepages for a container.  
To calculate affinity, Memory Manager gathers capacity of memory and pre-allocated hugepages per NUMA node except system and Kubenetes reserved capacity by Node Allocatable feature.
Then when pod admit is requested, Memory Manager checks resources availablity of each NUMA node and if possible, it reserves the resources internally.
- Node affinity of memory can be calcuated by below formulas.
  - Allocatable Memory of NUMA node = Total Memory of NUMA node - Hugepages - system reserved - kubernetes reserved.
  - Allocatable Memory of NUMA node >= Guaranteed memory for a container.

- Node affinity of hugepage can be calculated by the below formula.
  - Available huagpages of NUMA node = Total hugepages of NUMA node - reserved by system.
  - Available huagpages of NUMA node >= Guaranteed hugepages for a container.

##### Note
  - Node Allocatable feature offers "system-reserved" flag to reserve resources of CPU, memory, and ephemeral-storage for system services and kernel. At this time, kernel memory usage of container is accounted to system-reserved.
  - Node Allocatable feature offers "kube-reserved" flag to reserve resources for kublet and other kubernetes system daemons.
  - Node Allocatable feature reserves resources using cgroups that actually set a limitation of resource usage per NUMA node.
  - For this reason, it is hard to calculate allocatable memory at node level.
  - In alpha, Memory Manager reserves same amount of system-reserved kube-reserved memory per NUMA node.

#### Provide topology hint for Topology Manager
>Topology Manager defines HintProvider interface to take node affinity of resources from resource managers(i.e. CPU Manager and Device Manager).
Memory Manager implements the interface to calculate affinity of memory and hugepage and then provides calculated topology to Topology Manager.

##### Note

#### Isolate memory and hugepages for a container
>Cgroups cpuset subsystem is used to isolate memory and hugepages.
>When pod admit is requested with guaranteed QoS class, Memory Manager restricts a container to utilize a CPU and momory in same NUMA node.
>In other cases, Memory Manager also restricts memory access to a single NUMA node regardless of CPU affinity.

Consequently, Memory Manager guarantees that container's memory and hugepages are isolated to a single NUMA node.
 
#### New Interfaces

```go
package memory manager

type State interface {
  GetAllocatableMemory() uint64
  GetMemoryAssignments(containerID string) (memorytopology.MemoryTopology, bool)
  GetMachineMemory() memorytopology.MemoryTopology
  GetDefaultMemoryAssignments() ContainerMemoryAssignments
  SetAllocatableMemory(allocatableMemory uint64)
  SetMemoryAssignments(containerID string, mt memorytopology.MemoryTopology)
  SetMachineMemory(mt memorytopology.MemoryTopology)
  SetDefaultMemoryAssignments(as ContainerMemoryAssignments)
  Delete(containerID string)
  ClearState()
}

type Manager interface {
  Start(ActivePodsFunc, status.PodStatusProvider, runtimeService)
  AddContainer(p *v1.Pod, c *v1.Container, containerID string) error
  RemoveContainer(containerID string) error
  State() state.Reader
  GetTopologyHints(pod v1.Pod, container v1.Container) map[string][]topologymanager.TopologyHint
}

type Policy interface {
  Name() string
  Start(s state.State)
  AddContainer(s state.State, pod *v1.Pod, container *v1.Container, containerID string) error
  RemoveContainer(s state.State, containerID string) error
}
```

_Listing: Memory Manager and related interfaces (sketch)._

![memory-manager-interfaces](class_diagram.png)

_Figure: Memory Manager components._

![memory-manager-class-diagram](related_interfaces.png)

### Topology Manager changes

Topology Manager takes topology hints(bits that represents NUMA node) from resource managers such as CPU Manager and Device Manager. Then, it calculates the best affinity under given policy(i.e. preferred or strict) to determine pod admission. Likewise other resource managers, Memory Manager provides hints to Topology Manager.

Memory Manager implements 'Manager' interface, so that Topology Manager be able to add Memory Manager as hint provider at initialization sequence.

```go
package cm

func NewContainerManager(...) (ContainerManager, error) {
  ...
  cm.topologyManager.AddHintProvider(cm.memoryManager)
  ...
}
```

_Figure: Interfaces with topology manager_

![memory-manager-interface](interface.png)

### Internal Container Lifecycle changes

InternalContainerLifecycle interface defines following interfaces to manage container resources, PreStartContainer, PreStopContainer, PostStopContainer. In this cases, Memory Manager has to involve managing memory/hugepages when start/stop container.

Memory Manager manage memory with pod, container ID so these function needs that too.

```go
package cm

func (i *internalContainerLifecycleImpl) PreStartContainer(...) {
  ...
  err := i.memoryManager.AddContainer(pod, containerID)
  ...
}

func (i *internalContainerLifecycleImpl) PreStopContainer(...) {
  ...
  err := i.memoryManager.RemoveContainer(pod, containerID)
  ...
}

func (i *internalContainerLifecycleImpl) PostStopContainer(...) {
  ...
  err := i.memoryManager.RemoveContainer(pod, containerID)
  ...
}
```
### Feature Gate and Kubelet Flags

A new feature gate will be added to enable the Memory Manager feature. This feature gate will be enabled in Kubelet, and will be disabled by default in the Alpha release.

- Proposed Feature Gate:  
  `--feature-gate=MemoryManager=true`

This will be also followed by a Kubelet Flag for the Memory Manager Policy, which is described above. The `none` policy will be the default policy.

- Proposed Policy Flag:  
  `--memory-manager-policy=none|preferred|singleNUMA`

# Graduation Criteria

## Phase 1: Alpha (target v1.1x)
- Feature gate is disabled by default.
- Alpha Implimentation of Memory Manager based on SingleNUMA policy of Topology Manager

## Phase 2: Beta

- TBD

## GA (stable)

- TBD

[node-topology-manager]: https://github.com/kubernetes/enhancements/blob/dcc8c7241513373b606198ab0405634af643c500/keps/sig-node/0035-20190130-topology-manager.md
[cpu-manager]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/cpu-manager.md
[device-manager]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md
[hugepages]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190129-hugepages.md
[node-allocatable-feature]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-allocatable.md
[numa-issue]: https://github.com/kubernetes/kubernetes/issues/49964
[hugepage-issue]: https://github.com/kubernetes/kubernetes/issues/80716
