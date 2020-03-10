## SELinux and Containers

### Overview

In this section, we’ll cover the basics of SELinux and containers. SELinux policy prevents a lot of break out situations where the other security mechanisms fail. By default, podman processes are labeled with ```container_runtime_t``` and they are prevented from doing (almost) all SELinux operations.  But processes within containers do not know that they are running within a container.  SELinux-aware applications are going to attempt to do SELinux operations, especially if they are running as root. With RHEL, we write an SELinux policy that says that the container process running as ```container_runtime_t``` can only read/write files with the```container_file_t``` label.

#### !Namespaced

SELinux is not namespaced. Since we do not want SELinux aware applications failing when they run in containers, it was decided to make **libselinux** appear that SELinux is running to the container processes. In doing so, the **libselinux** library makes the following checks:

 * The ```/sys/fs/selinux``` directory is mounted read/write. 
 * The ```/etc/selinux/config``` file exists.

 If both of these conditions are not met, **libselinux** will report to calling applications that SELinux is disabled.  

##### Warmup Exercise 

Start by running a container that mounts ```/sys/fs/selinux``` as read-only then runs a command (```id -Z```)
that requires an SELinux enabled kernel. This is called bind mounting and is done with the ```-v``` option. Once the command runs, the container terminates.

~~~shell
$ podman run --rm -v /sys/fs/selinux:/sys/fs/selinux:ro {{RHEL_CONTAINER}} id -Z
~~~

Expected output:

~~~shell
id: --context (-Z) works only on an SELinux-enabled kernel
~~~

As mentioned above, our workaround to make the container aware that SELinux is running on
the host is to mount the ```/sys/fs/selinux``` directory as read/write and ```touch``` the SELinux
configuration file.

Give this a try and confirm the expected SELinux label is reported. You will need to run the
container as *root* with the `--privileged` flag.

~~~shell
# podman run -it --rm -v /sys/fs/selinux:/sys/fs/selinux:rw --privileged {{RHEL_CONTAINER}} bash
~~~

Enter the container and try an SELinux command. it should fail because both conditions have not been met.

~~~shell
{{CONTAINER_PROMPT}} id -Z

id: --context (-Z) works only on an SELinux-enabled kernel
~~~

Now touch the config file and try again. It should succeed. Note the SELinux label that is returned 
and the file label.

~~~shell
{{CONTAINER_PROMPT}} touch /etc/selinux/config
{{CONTAINER_PROMPT}} id -Z

unconfined_u:system_r:spc_t:s0

{{CONTAINER_PROMPT}} ls -lZ /etc/selinux/config

-rw-r--r--. 1 root root system_u:object_r:container_file_t:s0:c214,c1004 0 Mar  4 20:16 /etc/selinux/config

{{CONTAINER_PROMPT}}
~~~

Exit the container.

~~~shell
{{CONTAINER_PROMPT}} exit
~~~

#### Bind Mounts and Labeling

As you learned above, bind mounts allow a container to mount a directory on the host for general application usage. This lab will help you understand how selinux behaves in different bind mounting scenarios. 

On ```{{BASTION}}```, create a directory named ```/data```.

~~~shell
$ sudo mkdir /data 
~~~

Run bash in a container and volume mount the ```/data``` directory on {{BASTION}} to the ```/data``` directory in the container’s file system. 

~~~shell
$ podman run --rm -it -v /data:/data {{RHEL_CONTAINER}} bash
~~~

Notice the shell prompt changes to ```{{CONTAINER_PROMPT}}``` when you enter the container
namespace.

Did the mount succeed? How can you check? 

Run the ```df``` command to verify the volume is mounted to the ```/data``` directory.

~~~shell
{{CONTAINER_PROMPT}} df /data

Filesystem                      Size  Used Avail Use% Mounted on
/dev/mapper/rhel_rhel8b17-root   47G   17G   31G  35% /data
~~~

Now try to list the contents of ```/data```.   

~~~shell
{{CONTAINER_PROMPT}} ls /data

ls: cannot open directory /data: Permission denied
~~~

Try to create a file in the ```/data``` directory? The command will likely fail. Why? Hint: SELinux
is protecting the directory on the container host. 

~~~shell
{{CONTAINER_PROMPT}} date > /data/date.txt

bash: /data/date.txt: Permission denied

{{CONTAINER_PROMPT}} exit
~~~

The ```sealert``` tool will analyze and decode the ```audit.log``` file and make suggestions on how to remedy the problem.

To investigate, run the ```sealert``` tool on the client system.

~~~shell
$ sudo sealert -a /var/log/audit/audit.log 
~~~

There ```sealert``` tool produces a lot of information here. It should report that SELinux 
is preventing ```/usr/bin/bash``` from writing to the ```/data``` directory. It should
also report that to allow bash to have write access to ```/data```,
the SELinux label must be changed on that directory.

From the container host, examine the SELinux label of ```/data``` and note the type is ```default_t```. From
the discussion at the beginning of this lab, you know this is an SELinux label mismatch. Based on the 
default SELinux policy for containers, The label of the process does not match the label 
of the target directory.

~~~shell
$ ls -ldZ /data

drwxr-xr-x. root root unconfined_u:object_r:default_t:s0 /data
~~~

So let's take the suggestion of changing the label type of the ```/data``` directory to ```container_file_t```.

~~~shell
$ sudo chcon --type container_file_t /data
~~~

Confirm that ```/data``` is now correctly labeled.

~~~shell
$ ls -ldZ /data

drwxr-xr-x. root root unconfined_u:object_r:container_file_t:s0 /data
~~~

To allow this container to write to the ```/data``` , we also need to change the owner of the directory to ```lab-user``` on the client. Why is this?

~~~shell
$ sudo chown lab-user /data
~~~

From the client, check the permissions and labels again.

~~~shell
$ ls -ldZ /data
drwxr-xr-x. 2 lab-user root unconfined_u:object_r:container_file_t:s0 6 May  8 08:53 /data
~~~

Now run the container again and try to write into ```/data``` as you did above. Did the write succeed?

~~~shell
$ podman run --rm -it -v /data:/data {{RHEL_CONTAINER}} bash
~~~

~~~shell
{{CONTAINER_PROMPT}} ls /data
{{CONTAINER_PROMPT}}
{{CONTAINER_PROMPT}} date > /data/date.txt
~~~

Notice the directory permissions in the **container**. The owner is root (user namespaces in action).

~~~shell
bash-4.4# ls -ldZ /data

drwxr-xr-x. 2 root nobody unconfined_u:object_r:container_file_t:s0 6 May  8 18:39 /data
~~~

Time to exit the container.

~~~shell
{{CONTAINER_PROMPT}} exit
~~~

Finally, check the directory on the host. You should see
the file that was created with the correct ownership

~~~shell
$ ls -lZ /data
total 4
-rw-r--r--. 1 lab-user lab-user system_u:object_r:container_file_t:s0 29 May  8 09:30 date.txt
~~~

#### Private Mounts

Now you'll let podman create the SELinux labels. To change a label in the container context, you can add either of two suffixes ```:z``` or ```:Z``` to the volume mount. These suffixes tell podman to relabel file objects on the shared volumes. The ```:Z``` option tells podman to label the content with a private unshared label. 

Repeat the scenario above but instead add the ```:Z``` option to bind mount the ```/private``` directory then try to create a file in the ```/private``` directory from the container’s namespace.

First examine the default label for any new directory.

~~~shell
$ sudo mkdir /private
$ sudo chown lab-user /private
$ ls -dlZ /private

drwxr-xr-x. 2 lab-user root unconfined_u:object_r:default_t:s0 6 Apr  6 13:17 /private
~~~

Now run a container in the background that bind mounts ```/private``` using the ```:Z``` option.

~~~shell
$ podman run -d --name sleepy -v /private:/private:Z {{RHEL_CONTAINER}} sleep 9999

07c5aebd894182119668feddf4849d1f75bc5a81a84db222169e5f9b9efa625c
~~~

Examine the label again.

~~~shell
$ ls -dlZ /private

drwxr-xr-x. 2 lab-user root system_u:object_r:container_file_t:s0:c422,c428 6 Apr  6 13:17 /private
~~~

Note the addition of a unique Multi-Category Security (MCS) label (```c422,c428```) to the directory. SELinux takes advantage of MCS separation to ensure that the processes running in the container can only write to files with the same MCS Label.

Stop and remove the container.

~~~shell
$ podman stop sleepy
$ podman rm sleepy
~~~

#### Shared Mounts

Repeat the scenario above but instead add the ```:z``` option for the bind mount then try to create a file in the ```/shared``` directory from the container’s namespace. The ```:z``` option tells podman that two containers share the volume content. As a result, podman labels the content with a shared content label. Shared volume labels allow all containers to read/write content.

Create a directory named ```/shared``` and examine the label.

~~~shell
$ sudo mkdir /shared
$ sudo chown lab-user /shared
$ ls -dlZ /shared

drwxr-xr-x. 2 lab-user root unconfined_u:object_r:default_t:s0 6 Apr  6 14:09 /shared
~~~

Now run a container that bind mounts ```/shared``` using ```:z``` then create a file in ```/shared```.

~~~shell
$ podman run -it --rm --name sleepy -v /shared:/shared:z {{RHEL_CONTAINER}} bash
{{CONTAINER_PROMPT}} date > /shared/file01.txt
{{CONTAINER_PROMPT}} exit
~~~

On {{BASTION}}, notice the correct SELinux label on the shared directory.

~~~shell
$ ls -lZ /shared

-rw-r--r--. 1 lab-user lab-user system_u:object_r:container_file_t:s0 29 Apr  6 14:11 file01.txt
~~~

Repeat with a second container. It should succeed.

~~~shell
$ podman run -it --rm --name sleepier -v /shared:/shared:z {{RHEL_CONTAINER}} bash
{{CONTAINER_PROMPT}} date > /shared/file02.txt
{{CONTAINER_PROMPT}} exit
~~~

On {{BASTION}}, confirm the shared directory contains the files created by the containers.

~~~shell
$ ls -lZ /shared

-rw-r--r--. 1 lab-user lab-user system_u:object_r:container_file_t:s0 29 Apr  6 14:11 file01.txt
-rw-r--r--. 1 lab-user lab-user system_u:object_r:container_file_t:s0 29 Apr  6 14:15 file02.txt
~~~

#### Read-Only Containers

Imagine a scenario where an application gets compromised. The first thing the bad guy wants to do is to write an exploit into the container, so the next time the application starts up, it starts up with the exploit in place. If the container was read-only it would prevent leaving a backdoor in place and be forced to start the cycle from the beginning.

Container engines added a read-only feature but it presents challenges since many applications need to write to temporary directories like ```/run``` or ```/tmp``` and when these directories are read-only, the apps fail. Red Hat’s approach leverages ```tmpfs```. It's a nice solution to this problem because it eliminates data exposure on the host. As a recommended practice, run all applications in production in this mode and only allow write operations to known directories.

To experiment with this feature, run a read-only container and specify a few writable file systems using the ```--tmpfs``` option.

~~~shell  
$ podman run --rm -it --name tmpfs --read-only --tmpfs /run --tmpfs /tmp {{RHEL_CONTAINER}} bash
~~~

Now, try the following. What fails and what succeeds? Why?

~~~shell
{{CONTAINER_PROMPT}} mkdir /newdir

mkdir: cannot create directory '/newdir': Read-only file system

{{CONTAINER_PROMPT}} mkdir /run/newdir
{{CONTAINER_PROMPT}} mkdir /tmp/newdir
{{CONTAINER_PROMPT}} exit
~~~

We've covered a lot of ground here on Dan's favorite topic. You should feel good.







