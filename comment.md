` I would also like to see this misalignment happen naturally`
=> I think It can happen under below scenarios.
### Scenario A (Remote Hugepages Case)
- Container requests 2 exclusive cpus , 5GB Memory and 3x 1GB-Hugepages.
- deploy multiple setes of above container.

**[Idle State Case]**
![001](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-001.png)
When container is idle state, container may consume one or two hugepages.
In this case, there is no problem, there is no remote access of memory.
If containers always consume maximum size of hugepages, this case will not happen.  
(Kernel allocates hugepages when process allocates memory over `mmap`.
If process allocates less size of 1GB memory, kernel will allocate one hugepage.
If process allocates 1.5GB memroy, kernel will allocate two hugepages to process)

**[Busy State Case]**
![002](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-002.png)
When container is busy state, container may consume maximum quantity of hugepages.  
Here, it's three of 1GB-Hugepages.
In this case, if there is no extra hugepages on local numa node, container will comsume hugepages on another NUMA node.

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
Maybe you guys would say those above scenarios are not normal case or it's just a coner case.
But I want to say that it can happen beacuse cgroup sets only limits of memory regardless of memory capacity of NUMA node.

The point is `kubelet` does not consider memory capacity of NUMA node.
Topology Manager, CPU Manager and Device Manager (will) does it for cpu and devices(such as nic and gpu), but there is no consideration of Memory(including Hugepages) at all.

As @sjenning mentioned above, it seems that [numa_balancing](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/sect-virtualization_tuning_optimization_guide-numa-auto_numa_balancing) can cover most general cases(like idle state in scenarios).  
But it cannot **guarantee** local access of memory when there is no extra memory on a NUMA node.
To guarantee local access like other resources, Kubelet shoude schedule a container to a NUMA node and counting max 

The Openstack fully guarantee it, through `hw:numa_mem`property. See [here](https://docs.openstack.org/nova/rocky/admin/cpu-topologies.html#customizing-instance-numa-placement-policies) and [here](https://docs.openstack.org/nova/rocky/user/flavors.html#extra-specs).  
Beyond this they support 
