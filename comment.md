` I would also like to see this misalignment happen naturally`
=> I think It can happen under below scenarios.
### Scenario A (Remote Hugepages Case)
- Container requests 2 exclusive cpus , 5GB Memory and 3x 1GB-Hugepages.
- deploy multiple setes of above container.

**[Idle state]**
![001](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-001.png)
When container is idle state, container may consume one or two hugepages.
In this case, there is no problem, there is no remote access of memory.
If containers always consume maximum size of hugepages, this case will not happen.
(Kernel allocates hugepages when process allocates memory over `mmap`.
If process allocates less size of 1GB memory, kernel will allocate one hugepage.
If process allocates 1.5GB memroy, kernel will allocate two hugepages to process)

**[Busy state]**
![002](https://raw.githubusercontent.com/bg-chun/kep-draft/master/scenario-a-002.png)
When container is busy state, container may consume maximum quantity of hugepages.
Here, it's three of 1GB-Hugepages.
In this case, if there are no extra hugepages on local numa node, container will comsume hugepages on another NUMA node.


### Scenario B (Remote Memory Case)
