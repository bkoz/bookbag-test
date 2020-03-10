## Podman Basics

##### Overview

What is a pod?

A pod is a group of one or more containers, with shared storage/network, and a specification for how to run the containers. In this lab you'll be working at the container level. Since you are here to learn more about
container security, we assume you are comfortable with container basics. 

Podman (Pod Manager) is a fully featured container engine that is a simple daemon-less tool.  Podman provides a Docker-CLI comparable command line that eases the transition from other container engines and allows the management of pods, containers and images. Simply put: ```alias docker=podman```. Most ```podman``` commands can be run as a regular user, without requiring additional privileges. This is a significant security advantage.

#### Warmup Exercises

The container image that you will be using through out most of this lab is the [RHEL Universal Base Image (UBI)](https://access.redhat.com/containers/#/product/5c180b28bed8bd75a2c29a63). The UBI is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. Later on, you can read more about the [UBI](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html-single/getting_started_with_containers/index#using_red_hat_universal_base_images_standard_minimal_and_runtimes). 

##### Working with root and rootless containers.

Podman supports storing and running root and rootless containers. Effectively, each user manages
it's own containers.

The UBI container images should be loaded into the podman's local image storage for both root
and rootless (lab-user) usage. 

~~~shell
$ sudo gzip --decompress --stdout images/rhel-ubi8.tar.gz |sudo podman load
$ gzip --decompress --stdout images/rhel-ubi8.tar.gz |podman load
~~~

Confirm this using ```podman```.

~~~shell
$ sudo podman images
$ podman images

REPOSITORY                TAG      IMAGE ID       CREATED       SIZE
localhost/rhel-ubi8       latest   cc7efd763847   7 weeks ago   216 MB
~~~

The ```podman``` command may be run as **root** (privileged) or as a **root-less** (non-privileged)user. Let's start with a few warmup exercises. Note the container ID is returned when the container starts.

~~~shell
$ podman run --name=rootless -d {{RHEL_CONTAINER}} sleep 999

815dd74131decfed827b4087785e54b780eef12e44392ff1146c31179b29a855
~~~

~~~shell
$ podman ps

CONTAINER ID  IMAGE                       COMMAND    CREATED         STATUS             PORTS  NAMES
e05c3fc400eb  localhost/rhel-ubi8:latest  sleep 999  2 seconds ago   Up 2 seconds ago          rootless
~~~

~~~shell
$ sudo podman run --name=root -d {{RHEL_CONTAINER}} sleep 999 

815dd74131decfed827b4087785e54b780eef12e44392ff1146c31179b29a855
~~~

~~~shell
$ sudo podman ps

CONTAINER ID  IMAGE                       COMMAND    CREATED         STATUS             PORTS  NAMES
493da8f543de  localhost/rhel-ubi8:latest  sleep 999  43 seconds ago  Up 42 seconds ago         root
~~~

How to stop and remove a running container.

With grace.

~~~shell
$ podman stop rootless
$ podman rm rootless
~~~

~~~shell
$ sudo podman stop root
$ sudo podman rm root
~~~

With brute.

~~~shell
$ podman rm -f rootless
$ sudo podman rm -f root
~~~

##### podman top

Podman top can be used to display information about the running process of the container. Use it
to answer the following.

*Q:* What command is run when the container is run? How long has this container been running?

~~~shell
$ podman run --name=rootless -d {{RHEL_CONTAINER}} sleep 999
~~~

~~~shell
$ podman top -l args etime
~~~

Clean up.

~~~shell
$ podman rm -f rootless
~~~

##### Podman User Namespace Support

To observe user namespace support, you will run a rootless container
and observe the UID and PID in both the container and host namespaces.

Start by running a rootless container in the background. 

~~~shell
$ podman run --name sleepy -d {{RHEL_CONTAINER}} sleep 999
~~~

Next, run ```podman top``` to list the processes running in the 
container. Take note of the USER and the PID. The container process is running as
the ```lab-user``` user even though the container thinks it is ```root```. This is 
user namespaces in action. 

*Q:* What does the ```-l``` option do?

~~~shell
$ podman top -l
~~~

Next, on the host, list the same container process and take note of the USER and the PID.

~~~shell
$ ps -ef| grep sleep
~~~

Compare those answers to the same process running in the hosts
namespace.

The output from the host should resemble the following:

~~~shell
UID        PID  PPID  C STIME TTY          TIME CMD
lab-user  1701  1690  0 07:30 ?        00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 999
~~~

**=>** Take note of 2 important concepts from this example.

* The ```sleep``` process in the container is owned by ```root``` but
the process on the host is owned by ```lab-user```. This is
user namespaces in action. The **fork/exec** model used by podman 
improves the security auditing of containers. It allows an administrator to identify users
that run containers as root. Container engines that
use a ***client/server*** model can't provide this.

* The ```sleep``` process in the container has a PID of 1 but 
on the host the PID is **rootless** (a PID >1). This is
kernel namespaces in action.

##### Clean up

~~~shell
$ podman rm -f sleepy
~~~

##### Auditing containers

Take note of the ```lab-user``` UID.

~~~shell
$ sudo podman run --name sleepy --rm -it localhost/rhel-ubi8 bash -c "cat /proc/self/loginuid;echo"

1000
~~~

Configure the kernel audit system to watch the ```/etc/shadow``` file.

~~~shell
$ sudo auditctl -w /etc/shadow 2>/dev/null
~~~

Run a privileged container that bind mounts the host's file system then 
touches ```/etc/shadow```.

~~~shell
$ sudo podman run --privileged --rm -v /:/host localhost/rhel-ubi8 touch /host/etc/shadow
~~~

Examine the kernel audit system log to determine which user ran the malicious privileged container.

~~~shell
$ sudo ausearch -m path -ts recent -i | grep touch | grep --color=auto 'auid=[^ ]*'

type=SYSCALL msg=audit(04/30/2019 11:03:03.384:425) : arch=x86_64 syscall=openat success=yes exit=3 a0=0xffffff9c a1=0x7ffeee3ecf5c a2=O_WRONLY|O_CREAT|O_NOCTTY|O_NONBLOCK a3=0x1b6 items=2 ppid=6168 pid=6180 auid=lab-user uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=(none) ses=11 comm=touch exe=/usr/bin/coreutils subj=unconfined_u:system_r:spc_t:s0 key=(null) 
[lab-user@client ~]$
~~~

**=>** Try this at home using another container engine based on a client/server model and you 
will notice that the offending audit ID is reported as ```4294967295``` (i.e. an ```unsignedint(-1)```).
In other words, the malicious user is unknown.  

##### Extra Credit

A container administrator can make use *podman's* ```---uidmap``` option to force a range of UID's to be used. See
```podman-run(1)``` for details.

Run a container that maps 5000 UIDs starting at 100,000. This example maps uids 0-5000 in the container to the uids 100,000 - 104,999 on the host.

~~~shell
$ sudo podman run --uidmap 0:100000:5000 -d {{RHEL_CONTAINER}} sleep 1000

98554ea68dae250deeaf78d9b20069716e40eeaf1804b070eb408c9894b1df5a
~~~

Check the container.

~~~shell
$ sudo podman top --latest user huser | grep --color=auto -B 1 100000

USER   HUSER
root   100000
~~~

Check the host.

~~~shell
$ ps -f --user=100000

UID        PID  PPID  C STIME TTY          TIME CMD
100000    2894  2883  0 12:40 ?        00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 1000
~~~

Do the same beginning at 200,000.

~~~shell
$ sudo podman run --uidmap 0:200000:5000 -d {{RHEL_CONTAINER}} sleep 1000

0da91645b9c5e4d77f16f7834081811543f5d2c5e2a510e3092269cbd536d978
~~~

Check the container.

~~~shell
$ sudo podman top --latest user huser | grep --color=auto -B 1 200000

USER   HUSER
root   200000
~~~

Check the host.

~~~shell
$ ps -f --user=200000

UID        PID  PPID  C STIME TTY          TIME CMD
200000    3024  3011  0 12:41 ?        00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 1000
~~~

##### Challenge

Repeat the previous example but instead specify the user to be the **lab-user**.

Your ```top``` results should look like:

~~~sleep
$ sudo podman top -l user huser

USER   HUSER
1000   201000
~~~

##### References

[Pod concepts](https://kubernetes.io/docs/concepts/workloads/pods/pod/)