## Inspecting Content

Container images can easily be pulled from any public registry and run on a container host but is this good practice? Can you trust this image and what are its contents? A better practice would be to inspect the image before running it. 

#### Podman inspect

Use ```podman``` to inspect the {{RHEL_CONTAINER}} image. Examine the output to determine
who is the maintainer and what is the version of that image?

~~~shell
$ podman inspect {{RHEL_CONTAINER}}
~~~

#### Podman diff

The ```podman diff``` command can help understand the difference between a container image and
it's parent. 

~~~shell
$ podman diff --format=json {{RHEL_CONTAINER}}
~~~

#### Podman live mount

Next we’ll use the podman command to inspect a container’s filesystem by mounting it to the host.

First, launch a long running container in the background.

~~~shell
$ sudo podman run -d --name sleepy localhost/rhel-ubi8 sleep 9999

7ce0fdb9d8a9345e760fbcbf460d795a0a50cba1ac6e0ffe0e894c7a927cdcda
~~~

Next, enter the container's namespace, create some data then ```exit``` back to the host.

~~~shell
$ sudo podman exec -it sleepy bash
{{CONTAINER_PROMPT}} date >> /tmp/date.txt
{{CONTAINER_PROMPT}} exit
~~~

Next, mount the container. The host mount point should get displayed. 

~~~shell
$ sudo podman mount sleepy

/var/lib/containers/storage/overlay/<container-id>/merged
~~~

Using the directory returned from the above ```mount``` command, confirm the date.txt file 
exists in the mounted file system.

~~~shell
$ sudo cat /var/lib/containers/storage/overlay/<container-id>/merged/tmp/date.txt

Tue Apr 16 19:45:11 UTC 2019
~~~

How might you search this container's file system for all programs that are owned by root 
and have the SETUID bit set? 

~~~shell
$ sudo find /var/lib/containers/storage/overlay/<container-id>/merged -user root -perm -4000 -exec ls -ldb {} \;
~~~

Un-mount the container when you're finished.

~~~shell
$ sudo podman umount sleepy

<container-id>
~~~

Stop and remove the running container.

~~~shell
$ sudo podman stop sleepy
$ sudo podman rm sleepy
~~~

#### Working with Skopeo

Skopeo is an additional low level tool that can perform image operations on local or remote images. 
Give the example below a try.  

##### Inspection with Skopeo

First, make sure ```skopeo``` is installed.

~~~shell
$ sudo dnf install -y skopeo
~~~

Now try the following ```skopeo``` commands to inspect images from a remote registry. How
many layers does the rhel-ubi8 image contain?

~~~shell
$ skopeo inspect --tls-verify=false docker://{{SERVER_0}}/quayuser/rhel-ubi8
~~~

Expected output:

~~~shell
{
    "Name": "server.summit.lab/quayuser/rhel-ubi8",
    "Tag": "latest",
    "Digest": "sha256:8a5427d4c037826f4c69e268455d26327f2de158ffa1271a907fa357e9b93a3c",
    "RepoTags": [
        "latest"
    ], 
    ...
    ...
    ...
~~~

Skopeo can also copy images between two registries. See the ```skopeo(1)``` man page for details.



