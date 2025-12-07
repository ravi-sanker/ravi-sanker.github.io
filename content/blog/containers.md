+++
title = "Containers"
date = 2025-12-07
+++

# What is a Container?
A Container is an application with all its dependencies bundled together where it's "isolated" from the host machine it's running on. The advantage of this is that you can build the container once and run it anywhere. You don't get caught up in a dependency hell. Though containers have existed for a long time, Docker's ease of use popularised it. 

# Isolation
How isolated is a container from the host machine and the other containers running on the host? 

- A container should only be able to "see" its own file system, processes, users, network sockets etc.
- You should able to control the cpu usage, memory usage etc. of a container so that it doesn't go rogue and hog all the resources of the host, affecting other containers.

Note that as far as the host is concerned, the container is just another process. Unlike a VM, a container doesn't have its own kernel. It shares the same kernel as the host. So there's no real isolation between the host and a container. The host can see everything.

# It's all Linux, man. 

<img src="/containers/meme.png" alt="silly meme" width="500" style="display:block;margin:auto;"/>  

Turns out, you can implement a container with the isolation levels we talked about by stringing together a bunch of Linux commands. In the real world, most of the containers run on hosts that's running Linux. So understanding these Linux commands will get us closer to understanding containers.

## cgroups

Control groups limit the resources, such as memory, CPU, and network input/output, that a group of processes can use. Managing cgroups is all about reading and writing files under `/sys/fs/cgroup`.

```
root@8b9e89651172:/# ls /sys/fs/cgroup
cgroup.controllers      cpu.uclamp.min                   hugetlb.2MB.events.local   io.max               memory.swap.events
cgroup.events           cpu.weight                       hugetlb.2MB.max            io.pressure          memory.swap.high
cgroup.freeze           cpu.weight.nice                  hugetlb.2MB.numa_stat      io.prio.class        memory.swap.max
cgroup.kill             cpuset.cpus                      hugetlb.2MB.rsvd.current   io.stat              memory.swap.peak
cgroup.max.depth        cpuset.cpus.effective            hugetlb.2MB.rsvd.max       io.weight            memory.zswap.current
cgroup.max.descendants  cpuset.cpus.exclusive            hugetlb.32MB.current       memory.current       memory.zswap.max
cgroup.pressure         cpuset.cpus.exclusive.effective  hugetlb.32MB.events        memory.events        memory.zswap.writeback
cgroup.procs            cpuset.cpus.partition            hugetlb.32MB.events.local  memory.events.local  misc.current
cgroup.stat             cpuset.mems                      hugetlb.32MB.max           memory.high          misc.events
cgroup.subtree_control  cpuset.mems.effective            hugetlb.32MB.numa_stat     memory.low           misc.max
cgroup.threads          hugetlb.1GB.current              hugetlb.32MB.rsvd.current  memory.max           pids.current
cgroup.type             hugetlb.1GB.events               hugetlb.32MB.rsvd.max      memory.min           pids.events
cpu.idle                hugetlb.1GB.events.local         hugetlb.64KB.current       memory.numa_stat     pids.max
cpu.max                 hugetlb.1GB.max                  hugetlb.64KB.events        memory.oom.group     pids.peak
cpu.max.burst           hugetlb.1GB.numa_stat            hugetlb.64KB.events.local  memory.peak          rdma.current
cpu.pressure            hugetlb.1GB.rsvd.current         hugetlb.64KB.max           memory.pressure      rdma.max
cpu.stat                hugetlb.1GB.rsvd.max             hugetlb.64KB.numa_stat     memory.reclaim
cpu.stat.local          hugetlb.2MB.current              hugetlb.64KB.rsvd.current  memory.stat
cpu.uclamp.max          hugetlb.2MB.events               hugetlb.64KB.rsvd.max      memory.swap.current
```

Some of the files hold parameters that you can manipulate to define limits for the control group, and others communicate statistics about the current use of resources in the control group.

So what are the resources that you can restrict? This information is called a controller and each controller manages a type of resource that processes might want to consume. 

```
root@8b9e89651172:/# cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc
```

A process can only belong to one cgroup and this can be found in the `cgroup.procs` file. You can create a cgroup by simply creating a directory in `/sys/fs/cgroup/` and enforce limits by writing to the appropriate file. For example, if you want to set a limit on memory, you would update the `memory.max` file. It's obvious that you can create a hierarchy of cgroups with this design. The limits imposed by the parent cgroup trickles down to all its descendant cgroups. 

## chroot

It's important that the container can only see its own file system and not the host's or the files managed by the other containers. This can be achieved with the `chroot` command, which effectively changes the root directory of the process executing the command and all its children.

```
root@8b9e89651172:~# ls /
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@8b9e89651172:~# pwd
/root
root@8b9e89651172:~# mkdir new-root
root@8b9e89651172:~# chroot new-root/
chroot: failed to run command '/bin/sh': No such file or directory (os error 2)
```

`chroot` results in an error because `new-root` doesn't have anything. There's no `/bin/sh` file. So you need to have all the necessary binaries within your new root before you change the root. 

## namespaces

Apart from the file system, we now need to control what else the container can "see". This is achieved through namespaces. There are different types of namespaces in Linux:
- Cgroup
- IPC
- Network
- Mount
- PID
- Time
- User
- UTS

A process can only be in one namespace in each of these types. The simplest example is the UTS namespace. 

```
root@6710a85a535d:/# hostname
6710a85a535d
root@6710a85a535d:/# unshare --uts
# hostname
6710a85a535d
# hostname new-hostname
# hostname
new-hostname
# exit
root@6710a85a535d:/# hostname
6710a85a535d
```

In this example, you can see that you can change the hostname without it affecting the hostname of the host. This is basically the principle behind namespaces. The same principle applies for the other namespaces. For example, with the PID namespace you can control what processes the container can see and map the "true" process ID i.e. the process ID that the host sees to the process ID that the container sees. The implementation is slightly trickier than for UTS, but it's the same principle. 
**Namespaces give containers a set of resources that are independent of the host machine and of other containers.**

# References

[Container Security by Liz Rice](https://www.oreilly.com/library/view/container-security/9781492056690/)
