` I would also like to see this misalignment happen naturally`
=> I think it can happen under below scenarios, it's a kind of resource exhausted case.

### Scenario A (Remote Hugepages Case)
- Container requests 2 exclusive cpus , 5GB Memory and 3x 1GB-Hugepages.
- deploy multiple sets of above container.

**[Idle State Case]**
![001](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-001.png)
When container is idle state, container may consume one or two hugepages.
In this case, there is no problem, there is no remote access of memory.
If containers always consume maximum size of hugepages, this case will not happen.  
(Kernel allocates hugepages when process allocates memory over `mmap`.
If process allocates less size of 1GB memory, kernel will allocate one hugepage.
If process allocates 1.5GB memory, kernel will allocate two hugepages to process)

**[Busy State Case]**
![002](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-002.png)
When container is busy state, container may consume maximum quantity of hugepages.  
Here, it's three of 1GB-Hugepages.
In this case, if there is no extra hugepages on local numa node, container will consume hugepages on another NUMA node.

### Scenario B (Remote Memory Case)
- Container Blue requests 2 exclusive cpus, 95GB Memory and 3x 1GB-Hugepages.
- Container Red requests 2 exclusive cpus , 5GB Memory and 3x 1GB-Hugepages.

**[Idle State Case]**
![003](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-003.png)  
In this scenario, container blue can be such kind of In-Memory Database(redis).  
Or it can be multiple containers that sharing cpus.  
When container red is idle state, there will be no remote access of memory.

**[Busy State Case]**
![004](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-004.png)  
But when container red is busy state, there will be remote access of memory.

![005](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-005.png)
Maybe, this worstest case can happen, this case will reduce performance of both containers.

## My conclusion
Maybe you guys would say that above scenarios are not normal case or it's just a corner case.
But I want to say that it can happen because cgroup sets only limits of memory regardless of memory capacity of NUMA node.

The point is `kubelet` does not consider memory capacity of NUMA node.
Topology Manager, CPU Manager and Device Manager (will) does it for cpu and devices(such as nic and gpu), but there is no consideration of Memory(including Hugepages) at all.

As @sjenning mentioned above, it seems that [numa_balancing](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/sect-virtualization_tuning_optimization_guide-numa-auto_numa_balancing) can cover most general cases(like idle state in scenarios).  
But it cannot **guarantee** local access of memory for container.  
Without `guaranteeing local access of memory`, we cannot guarantee performance of DPDK containers.  
To guarantee local access of memory, kubelet should schedule container to a NUMA node based on container's memory request.  
Then enforce container to consume memory on a NUMA node using a `cupset.mems`.  
Below image shows a concept of scheduling.  

![concept-of-scheduling](https://raw.githubusercontent.com/bg-chun/kep-draft/master/concept-of-scheduling.png)


Otherwise, Openstack which is used to run VNF(DPDK) fully guarantee it, through `hw:numa_mem`property, see [here](https://docs.openstack.org/nova/rocky/admin/cpu-topologies.html#customizing-instance-numa-placement-policies) and [here](https://docs.openstack.org/nova/rocky/user/flavors.html#extra-specs).  
Moreover Openstack also support that VM takes certain amount of memory from multiple NUMA nodes, see table 3.2 [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_openstack_platform/7/html/instances_and_images_guide/ch-manage_instances#section-update-flavor-metadata).  
It seems that Libvirt driver support it, but i'm not sure, i was just an one of users of Openstack.

Example
``` 
Example when the instance has 8 vCPUs and 4GB RAM:
    hw:numa_nodes=2
    hw:numa_cpus.0=0,1,2,3,4,5
    hw:numa_cpus.1=6,7
    hw:numa_mem.0=3  //Mapping 3 GB of RAM to NUMA node 0
    hw:numa_mem.1=1  //Mapping 1 GB of RAM to NUMA node 1. 
```
