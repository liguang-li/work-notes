# Cgroups

Written by Paul Menage <menage@google.com> based on [Documentation/cgroup-v1/cpusets.txt](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)

## Control Groups
* What are cgroups?
Control Groups provide a mechanism for aggregatng/partitioning sets of tasks, and all their future children into hierarchical groups with specialized behaviour.

	* A *Group* associates a set of tasks with a set of parameters for one or more subsystems.
	* A *subsystem* is a module that makes use of the task grouping facilities provided by cgroups to treat groups of tasks in particular ways. A subsystem is typically a "resource controller" that schedules a resource or applies per-cgroup limits, but it may be anything that wants to act on a group of processes, e.g. a virtualization subsystem.
	* A *hierarchy* is a set of cgroups arranged in a tree, such that every task in the system is in exactly one of the cgroups in the hierarchy, and a set of subsystems; each subsystem has system-specific state attached to each cgroup in the hierarchy. Each hierarchy has an instance of the cgroup virtual subsystem associated with it.

At any one time there may be multiple active hierarchies of task cgroups. Each hierarchy is a partition of all tasks in the system.

## Why are cgroups needed?
There are multiple efforts to provide process aggregations in the Linux kernel, mainly for resource-tracking purposes. Such efforts include cpusets etc. These all require the basic notion of a grouping/partitioning of processes, with newly forked process ending up in the same group as their parent process.

The kernel cgroup patch provides the minimum essential kernel mechanisms required to efficiently implement such groups. It has minimal impact on the system fast paths, and provides hooks for specific subsystem such as cpusets to provide additional behaviour as desired.

提供了多层次结构支持，以允许在不同的子系统中将任务划分为不同的cgroup的情况
Multiple hierarchy support is provided to allow for situations where the division of tasks into cgroups is distinctly different for different subsystems.

At one extreme, each resource controller or subsystem could be in a separate hierarchy; at the other extreme, all subsystems would be attached to the same hierarchy.

## How the cgroups implemented?
Control Groups extends the kernel as following:
* Each task in the system has a reference-counted pointer to a css_set
* A css_set contains a set of reference-counted pointers to cgroup_subsys_state objects, one for each cgroup subsystem registered in the system. There is no direct link from a task to the cgroup of which it's a member in each hierarchy, but this can be determined by following pointers through the cgroup_subsys_state objects. This is because accessing the subsystem state is something that's expected to happen frequently and in performance-critical code, whereas operations that require a task's actual cgroup assignments are less common. A linked list runs through the cg_list field of each task_struct using the css_set, anchored at css_set->tasks.
* A cgroup hierarchy filesystem can be mounted for browsing and manipulation from user space.
* Can list all the tasks (by PID) attached to any cgroup.

The implemention of cgroups requires a few, simple hooks into the rest of the kernel, none in the performance-critical paths:
* in init/main.c, to initialize the root cgroups and initial css_set at system boot.
* in fork/exit, to attach and detach a task from its css_set.

In addition, a new file system of type "cgroup" may be mounted, to enable browsing and modifying the cgroups presently known to the kernel. Which mounting a cgroup hierarchy, you may specify a comma-separted of list of subsystems to mount as the filesystem mount options. By default, mounting the cgroup filesyetem attempts to mount a hierarchy containing all the registered subsystems.

If an active hierary with exactly the same set of subsystems already exists, it will be reused for the new mount. If no existing hierarchy matches, and any of the requested subsystems are in use in an existing hierarchy, the mount will fail with -EBUSY. Otherwise, a new hierarchy is activated, associated with the requested subsystems.

It's not currently possible to bind a new subsystem to an active cgroup hierarchy, or to unbind a subsystem from an active cgroup hierarchy. This may be possible in future, but is fraught with nasty error-recovery issues.

When a cgroup filesystem is unmounted, if there are any child cgroups created below the top-level cgroup, that hierarchy will remain active even through unmounted; if there are no child cgroups then the hierarchy will be deactivated.

No new system calls are added for cgroups - all support for querying and modifying cgroups is via this cgroup file system.

Each task under /proc has an added file named 'cgroup' displaying, for each active hierarchy, the subsystem names and the cgroup name as the path relative to the root of the cgroup file system.
~~~
xuan@xuan-K43SJ:~$ cat /proc/self/cgroup 
12:rdma:/
11:memory:/
10:blkio:/
9:net_cls,net_prio:/
8:cpuset:/
7:freezer:/
6:pids:/user.slice/user-1000.slice/user@1000.service
5:cpu,cpuacct:/
4:perf_event:/
3:devices:/user.slice
2:hugetlb:/
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/gnome-terminal-server.service
0::/user.slice/user-1000.slice/user@1000.service/gnome-terminal-server.service
~~~shell

Each cgroup is represented by a directory in the cgroup file system containing the following file describing that cgroup:
* tasks: list of tasks (by PID) attached to that cgroup. This list is not guaranteed to be sorted. Writing a thread ID into this file moves the thread into this cgroup.

* cgroup.procs: list of thread group IDs in the cgroup. This list is not guranteed to be sorted or free of duplicated TGIDs, and userspace should sort/uniquify the list if this property is required. Writing a thread group ID into this file moves all threads in that group into this cgroup.

* notify_on_release flag: run the release agent on exit?

* release_agent: the path to use for release notifications (exists in the top cgroup only).

New cgroups are created using mkdir system call or shell command. The properties of a cgroup, such as its flags, are modified by writing to appropriate file in that cgroups directory.

* The named hierarchical structure of nested cgroups allows partitioning a large subsystem into nested, dynamically changeable, "soft-partitions".
* The attachment of each task, automatically inherited at fork by any children of that task, to a cgroup allows organizing the work load on a system into related sets of tasks. A task may re-attached to any other cgroup, if allowed by the permissions on the necessary cgroup file system directories.
* When a task is moved from one cgroup to another, it gets a new css_set pointer - if there's an already existing css_set with desired collection of cgroups then that group is reused, otherwise a new css_set is allocated. The appropriate existing css_set is located by looking into a hash table.
* To allow access from a cgroup to the css_sets (and hence tasks) that comprise it, a set of cg_group_link objects from a lattice. each cg_cgrop_link is linked into a list of cg_cgroup_links for a single cgroup on its cgrp_link_list field, and a list of cg_cgroup_links for a single css_set on its cg_link_list.

The use of a Linux virtual file system (vfs) to represent the cgroup hierarchy provides for a familiar permission and name space for cgroups, with a minimum of additional kernel code.

## What does notify_on_release do?
If the notify_on_release flag is enabled (1) in a cgroup, then whenever the last task in the cgroup leaves (exists or attaches to some other cgroup) and the last child cgroup of that cgroup is removed, then the kernel runs the command specified by the contents of the "release_agent" file in that hierarchy's root directory, supporting the pathname (relative to the mount point of the cgroup file system) of the abandoned cgroup. This enables automatic removal of abandoned cgroups. The default value of notify_on_release in the root cgroup at system boot is disabled (0). The default value of other cgroups at creating is the current value of their parents' notify_on_release settings. The default value of a cgroup hierarchy's release_agent path is empty.

## What does clone children do?
The flag only affects the cpuset controller. If the clone_children flag is enabled (1) in a cgroup, a new cpuset cgroup will copy its configuration from the parent during initialization.

# How do i use cgroups?
1. mount -t tmpfs cgroup_root /sys/fs/cgroup
2. mkdir /sys/fs/cgroup/cpuset
3. mount -t cgroup -o cpuset cpuset /sys/fs/cgroup/cpuset
4. mkdir's or write's
5. Attach a task to the new cgroup by writing its PID to the /sys/fs/cgroup/cpuset/tasks
6. fork, exec or clone jobs tasks

**While remounting cgroups is currently supported, it's not recommand to use it**
* Remounting allows changing bound subsystems and release_agent.
* Rebinding is hardly useful as it only works when the hierarchy is empty and release_agent itself should be replace with conventional fsnotify.
* The support of remounting will be removed in the future.

# Kernel API
## Overview
Each kernel subsystem that wants to hook into the generic cgroup system needs to create a cgrout_subsys object. This contains various methods, which are callbacks from the cgroup system, along with a subsystem ID which will be assigned by the cgroup system.

* subsys_id: a unique array index for the subsystem, including which entry in cgroup->subsyst[] this subsystem should be managing.
* name: should be initialized to a unique subsystem name.
* early_init: indicate if the subsystem needs early initialization at system boot.

Each cgroup object created by the system has an array of pointers, indexed by the subsystem ID; this pointer is entirely managed by the subsystem; the generic cgroup code will never touch this pointer.

## Synchronization
A global mutex, cgroup_mutex, used by the cgroup system.

Accessing a task's cgroup pointer may be done in the following ways:
* while holding cgroup_mutex;
* while holding the task's alloc_lock (via task_lock())
* inside an rcu_read_lock() section via rcu_dereference()

## Subsystem API
* add an entry in linux/cgroup_subsys.h
* define a cgroup_subsys object called <name>_cgrp_subsys

struct cgroup_subsys_state * css_alloc(struct cgroup * cgrp)
(cgroup_mutex held by the caller)

Called to allocate a subsystem state object for a cgroup. The subsystem should allocate its subsystem state object for the passed cgroup, returning a pointer to the new object on success or a ERR_PTR() value. On success, the subsystem pointer should point to a structure of type cgroup_subsys_state (typically embedded in a larger subsystem-specific object), which will be initialized by the cgroup system.

Note that this will be called at initialization to create the root subsystem state for this subsystem; this case can be identified by the passed cgroup object having a NULL parent (since it's the root of the hierarchy) and may be an appropriate place for initialization code.

int css_online(struct cgroup * cgrp)
void css_offline(struct cgroup * cgrp)
void css_free(struct cgroup * cgrp)

int can_attach(struct cgroup * cgrp, struct cgroup_taskset * tset)
Called prior to moving one or more tasks into a cgroup; if the subsystem return an error, this will abort the attach operation.

If there are multiple tasks in the taskset, then:
* it's guaranteed that all are from the same thread group
* @tset contains all tasks from the thread group whether or not they're switching cgroups
* the first task is leader

~~~
void css_reset(struct cgroup_subsys_state *css)

void cancel_attach(struct cgroup *cgrp, struct cgroup_taskset *tset)
Called when a task attach operation has failed after can_attach() has succeeded.
~~~c

void attach(struct cgroup *cgrp, struct cgroup_taskset *tset)
Called after the task has been attached to the cgroup, to allow any post-attachment activity that requires memory allocations or blocking.

void bind(struct cgroup *cgrp)
Called when a cgroup subsystem is rebound to a different hierarchy and root cgroup.
