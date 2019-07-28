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
      - [New Interfaces](#new-interfaces)
    - [Support container isolation of huge pages](#support-container-isolation-of-huge-pages)
    - [Support reserve huge pages for system on Node Allocatable feature](#support-reserve-huge-pages-for-system-on-node-allocatable-feature)
    - [cAdviser changes](#cAdviser-changes)
    - [Topology Manager changes](#topology-manager-changes)
    - [Internal Container Lifecycle changes](#internal-container-lifecycle-changes)
    - [Feature Gate and Kubelet Flags](#feature-gate-and-kubelet-flags)
- [Graduation Criteria](#graduation-criteria)
  - [Phase 1: Alpha (target v1.1x)](#phase-1-Alpha-target-v11x)
  - [Phase 2: Beta](#phase-1-Beta)
  - [GA (stable)](#ga-stable)
- [Limitations and Challenges](#Limitations-and-Challenges)
- [Appendix A](#Appendix-A)

# Overview

NUMA Awareness is well known for a solution to boost performance for diverse use cases including DPDK and Database. Kubernetes support NUMA related features for NUMA sensitive containers.
The features(CPU Manager, Device Manager, Topology Manager) treat CPU, Device and it's NUMA topology, but there's no feature cover Memory and Hugepages. It can cause inter-NUMA node communication for Memory access and result Memory latency and performance issue.

The Memory Manager is proposed for a solution deploying Pod and Containers with Guaranteed QoS class of Memory and Hugepages under NUMA awareness.

# Motivation

## Related Features

- [Node Topology Manager][node-topology-manager] is a feature that collects topology hint from various hint providers to calculate socket affinity for a container. Topology Manager judge container whether admit or not under configured policy and socket affinity.

- [CPU Manager][cpu-manager] is a feature provides a solution for CPU Pinning based on cgroups cpuset subsystem, it also offer topology hint to Topology Manager.

- [Device Manager][device-manager] is a one of features provides topology hint for Topology Manager. The main objectivity of Device Manager is allow vendors to adverise their resources to kubelet.

- [Hugepages][hugepages] is a feature allows container to consume pre-allocated hugepages with pod isolation of huge pages.

- [Node Allocatable Feature][node-allocatable-feature] is a feature helps to reserve compute resources to prevent resource starvation. The kube-reserved and system-reserved is used to reserve resources for kubelet and system(OS, etc). Now following resources are supported to reserve: cpu, memory, ephemral storage.

## Related issues

- [Hardware topology awareness at node level (including NUMA)][numa-issue]

## Goals

- Guarantee alignment of resources for isolating container's Memory and Hugepages with CPU and I/O devices.

- Provide topology hint for Memory and Hugepages to Topology manager.

- Support container isolation of hugepages

- This proposal only focus on Linux based system.

- This proposal only cover pre-allocated hugepage.

## Non-Goals

- Support multi size of hugepages together

## User Stories

### Story 1 : Networking Acceleration using DPDK

- DPDK(Data Plane Development Kit) is set of libraries to accelerate packet processing on userspace. DPDK requires dedicated resources and alignment of resources under NUMA awareness. There should be a way to gurantee reserve Memory and Hugepages with other computing resources on desinated Socket for containerized VNF using DPDK.

### Story 2 : Database

- Databases(Oracle, PostgreSQL, MySQL, MongoDB, SAP) consumes massive Memory and Hugepages, in order to reduce memory access latency and improve performance dramatically resources(CPU, Memory, Hugepages, I/O Devices) should be aligned.

# Proposal

## Proposed Changes

### New Component: Memory Manager

A new component of kubelet enable NUMA-awareness for memory and hugepage usages for a pod. Manages memory / hugepage resource of host by node. When a pod deployment request comes in, it checks the available memory / hugepage per numa node and provide the topology manager a topology hint. It manages the memory / hugepage assigned to the container and updates the available resources for each node when the container is deployed / deleted. These component features can be turned off through feature flag settings.

#### New Interfaces

```go
package memory manager

type State interface {
  GetAllocatableMemory() uint64
  GetMemoryAssignments(containerID string) (MemorySet, bool)
  GetDefaultMemoryAssignments() ContainerMemoryAssignments
  GetMemoryPool() MemorySet
  SetAllocatableMemory(allocatableMemory uint64)
  SetMemoryAssignments(containerID string, ms MemorySet)
  SetDefaultMemoryAssignments(as ContainerMemoryAssignments)
  SetMemoryPool(memorySet MemorySet)
  Delete(containerID string)
  ClearState()
}

type Manager interface {
  Start(ActivePodsFunc, status.PodStatusProvider, runtimeService)
  AddContainer(p *v1.Pod, c *v1.Container, containerID string) error
  RemoveContainer(containerID string) error
  State() state.Reader
  GetTopologyHints(pod v1.Pod, container v1.Container) []topologymanager.TopologyHint
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

### Support container isolation of huge pages

 Under Pod isolation of hugepage there would be a competition to consume Hugepage between containers. As an concrete result, when two containers requests one of 1GB Hugepages, kubelet sets limit on Pod's cgroup with 2GB.
 In this case, if a single container consumes more than 1GB Hugepages another on not able to consume hugepage at all, to avoid this case container isolation of hugepage shoud be suported.


### Support reserve huge pages for system on Node Allocatable feature

- Do not know how much hugepage OS and kubelet use. Thus, To work the memory manager reliably, hugepages should be reserved for pod. An example of OS or kubelet using hugepage is OVS-DPDK. OVS-DPDK is very improtant for VNF. To ensure hugepage for pod, we need to add a hugepage flag to the Node Allocatble feature.

### cAdviser changes

### Topology Manager changes

Topology Manager takes hints from cpu manager, device manager. After that, calculate best affinity with policy and used to determine if the pod can admit. Likewise, Memory Manager provide hints to Topology Manager.

Memory Manager implement 'Manager' interface. So Topology Manager add Memory Manager as hint provider at initializing container manager.

```go
package cm

func NewContainerManager(...) (ContainerManager, error) {
  ...
  cm.topologyManager.AddHintProvider(cm.memoryManager)
  ...
}
```

Provided hint from Memory Manager is affinity of NUMA nodes(bits) like other managers. So there will be few or no changes to the topology manager.

_Figure: Interfaces with topology manager_

![memory-manager-interface](interface.png)

### Internal Container Lifecycle changes

InternalContainerLifecycle interface has 3 functions, PreStartContainer, PreStopContainer, PostStopContainer. In this cases, Memory Manager has to involve managing memory/hugepages when start/stop container.

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
  `--memory-manager-policy=none|strict`

# Graduation Criteria

## Phase 1: Alpha (target v1.1x)

- container level hugepage isolation

- new topology hint provider for memory and hugepage

- add new flag for hugepages on node allocatable feature(host에서 ovs-dpdk가 사용할 hugepage많큼 제외시켜줘야함,)

## Phase 2: Beta

- cAdvisor (세원님이 개발한 코드가 cAdvisor에 기여)

- container runtime이 hugepage 지원한다면 integration? 근대 runc runtime을 누군가가 개선 해 줘야하는대 이걸 누가 해주려나

- Integration Alpha impliments of hugepage to existing hugepage advertising impliments. (현재 스케쥴러에게 hugepage정보 보내주는건 Redhat + intel이 hugepage KEP로 개발 해놓은 상황이고 Alpha구현에선 여길 손 안댐, Beta에선 기존에 Redhat이 개발해놓은 부분을 우리구현으로 대체가능할것으로 보이며 선행조건으로 세원님이 개발한 코드가 cAdvisor에 기여되어야 할듯)

## GA (stable)

#

# Limitations and Challenges

- 한계와 도전 둘중 하나만 해도 괜찮을거 같은대

- 완벽한 메모리 관리는 어려움. (os/kubelet reserved만큼 여유를 두면 상관없지만 이는 손해임) => offset 두는 쪽으로

- NUMA 노드 별 메모리 관리에 대한 보장 => offset 두어서 안전하게 관리한다는걸 어필해보자

- 외부 요인에 의한 완벽한 Hugepages 관리가 어렵다. => 삭제 node allocatable featuree 개선시 해결됨

- 컨테이너의 자원이 하나의 NUMA 노드에 대한 isolation만 가능하다.
  (단일 컨테이너가 여러 Node에서 자원을 할당받으려면 어떻게해야할까..)
  베타로 넣어놓고 못하면 인텔 처럼 빤스런?

[node-topology-manager]: https://github.com/kubernetes/enhancements/blob/dcc8c7241513373b606198ab0405634af643c500/keps/sig-node/0035-20190130-topology-manager.md
[cpu-manager]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/cpu-manager.md
[device-manager]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md
[hugepages]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190129-hugepages.md
[node-allocatable-feature]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-allocatable.md
[numa-issue]: https://github.com/kubernetes/kubernetes/issues/49964
