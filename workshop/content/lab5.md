## Isolation

### Overview
Containers provide a certain degree of process isolation via kernel namespaces. In this lab, we’ll examine the capabilities of a process running in a containerized namespace. In this lab we'll look at Linux capabilities as it relates to containers. Capabilities can be used to grant a particular or subset of root privileges to a process or container. It would be worth taking a few minutes to read this [blog post](http://rhelblog.redhat.com/2016/10/17/secure-your-containers-with-this-one-weird-trick/) before beginning this lab. 

### Capabilities

Capabilities are distinct units of privilege that can be independently enabled or disabled.

Start by installing the kernel header files.

~~~shell
$ sudo dnf -y install kernel-headers
~~~

Next exam the process capabilities header file. 

~~~shell
$ vim -R /usr/include/linux/capability.h
~~~

Turn on **line numbering** in vim.

~~~shell
:set number
~~~

The capability bitmask definitions begin at line 107 with a short description followed by the
bit position. For example, the first capability is ```CAP_CHOWN``` (bit 0). 

~~~shell
103 /**
104  ** POSIX-draft defined capabilities.
105  **/
106 
107 /* In a system with the [_POSIX_CHOWN_RESTRICTED] option defined, this
108    overrides the restriction of changing file ownership and group
109    ownership. */
110 
111 #define CAP_CHOWN            0
~~~

Examine the capabilities bitmask of a **rootless** process running on the host. The bitmask returned is ```0x0```
meaning this process does not have any capabilities. In particular, notice that bit 0 is not
set (```CAP_CHOWN```) which means this process can not execute ```chown``` or ```chgrp``` on a file that it does not own.

~~~shell
$ grep CapEff /proc/self/status

CapEff:	0000000000000000
~~~

Now examine the capabilities bitmask of a **root** process on the host. Notice that all 38 capability bits are set indicating this process has a full set of capabilities. In a later exercise, you'll discover that a container
running as **root** has a filtered set of capabilities by default but this can be changed at run time.

~~~shell
$ sudo grep CapEff /proc/self/status

CapEff:	0000003fffffffff
~~~

The ```capsh``` and ```pscap``` commands provide a human readable output of the capabilities bitmask. Try it out!

~~~shell
$ capsh --decode=01fffffffff

0x0000001fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,35,36
~~~

##### Exploring the capabilities of containers.

 A non-null CapEff value indicates the process has capabilities. Take note that the capabilities of a container are less than what a root process has running on the host.

Start by running a rootless container as ```--user=32767``` and look at it’s capabilities.

~~~shell
$ podman run --rm -it --user=32767 {{RHEL_CONTAINER}} grep CapEff /proc/self/status

CapEff:	0000000000000000
~~~

Next run a rootless container as ```--user=0``` and look at it’s capabilities. Note that it's
capabilities are filtered from that of a root process on the host.

~~~shell
$ podman run --user=0 --rm -it {{RHEL_CONTAINER}} grep CapEff /proc/self/status

CapEff:	00000000a80425fb
~~~

Now run a container as privileged and compare the results to the previous exercises. What conclusions can you draw?

~~~shell
$ sudo podman run --user=0 --rm -it --privileged {{RHEL_CONTAINER}} grep CapEff /proc/self/status

CapEff: 0000003fffffffff
~~~

#### Podman top

Run a container in the background that runs for a few minutes.

~~~shell
$ podman run --user=0 --name sleepy -it -d {{RHEL_CONTAINER}} sleep 999
~~~

Use ```podman top``` to examine the container's capabilities.

~~~shell
$ podman top sleepy capeff

EFFECTIVE CAPS
AUDIT_WRITE,CHOWN,DAC_OVERRIDE,FOWNER,FSETID,KILL,MKNOD,NET_BIND_SERVICE,NET_RAW,SETFCAP,SETGID,SETPCAP,SETUID,SYS_CHROOT
~~~

Try out the following very useful shortcut (```-l```). It tells ```podman``` to act on the latest container it is aware of.

~~~shell
$ podman top -l capeff
~~~

Podman **top** has many additional format descriptors you can check out.

~~~shell
$ podman top -h
~~~

#### Capabilities Challenge #1

How could you determine which capabilities podman drops from a **root** process running in a container? 
If you need a hint, one solution is presented below.

One approach would be to use your favorite binary calculator (```bc```) to compare the CapEff difference between a host process (0x3fffffffff) and a containerized process (0xa80425fb) then use ```capsh``` to decode it.

~~~shell
$ echo 'obase=16;ibase=16;3FFFFFFFFF-A80425FB' | bc
3F57FBDA04

$ capsh --decode=3F57FBDA04 

0x0000003f57fbda04=cap_dac_read_search,cap_linux_immutable,cap_net_broadcast,cap_net_admin,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_lease,cap_audit_control,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
~~~

#### Finer grained capabilities.

Next, run the container as root but drop all capabilities.

~~~shell
$ podman run --rm -ti --user 0 --name temp --cap-drop=all {{RHEL_CONTAINER}} grep CapEff /proc/self/status

CapEff:	0000000000000000
~~~

Now, run the container as root but add all capabilities.

~~~shell
$ podman run --rm -ti --user 0 --name temp --cap-add=all {{RHEL_CONTAINER}} grep CapEff /proc/self/status

CapEff: 0000003fffffffff
~~~

#### Capabilities Challenge #2

Suppose a container had a legitimate reason to change the date (ntpd, license testing, etc) How would you allow a container to change the date on the host? What capabilities are needed to allow this? One solution is below.

To allow a container to set the system clock, the ```sys_time``` capability must be added. 

Run a container, save the date then try to change the date. It should fail because a non-root
user can not add a capability that requires root privileges like setting the system clock. 

~~~shell
$ podman run --rm -ti --user 0 --name temp --cap-add=sys_time {{RHEL_CONTAINER}} bash

{{CONTAINER_PROMPT}} savethedate=$(date)
{{CONTAINER_PROMPT}} date -s "$savethedate"

date: cannot set date: Operation not permitted
Mon Apr  8 21:45:24 UTC 2019

{{CONTAINER_PROMPT}} exit
~~~

Now try adding the ```sys_time``` capability then try setting the date again. It should succeed.

~~~shell
$ sudo podman run --rm -ti --user 0 --name temp --cap-add=sys_time {{RHEL_CONTAINER}} bash

{{CONTAINER_PROMPT}} savethedate=$(date)
{{CONTAINER_PROMPT}} date -s "$savethedate"

Mon Apr  8 21:46:18 UTC 2019

{{CONTAINER_PROMPT}} exit
~~~
