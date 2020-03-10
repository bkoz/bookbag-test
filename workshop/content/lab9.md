## WORK IN PROGRESS: Secure Builds

### Scratch builds

Paragraph on the benefits of building from scratch.

Explain what `buildah unshare` does. 

~~~shell
$ buildah unshare

[root@bastion 0 ~]#
~~~

The `#` prompt indicates `uid=0` in the container
namespace but not `uid=0` on the host. Can't beleive your eyes? Try a `root` command
like `cat /etc/shadow`.

~~~shell
[root@bastion 0 ~]# cat /etc/shadow
cat: /etc/shadow: Permission denied
[root@bastion 1 ~]#
~~~

~~~shell
[root@bastion 0 ~]# buildah from scratch

[root@bastion 0 ~]# scratchmnt=$(buildah mount working-container)

[root@bastion 0 ~]# echo $scratchmnt

[root@bastion 0 ~]# ls -R $scratchmnt
~~~

Note that the root of the scratch container is EMPTY!

Start by installing the `redhat-release` package. 

~~~shell
[root@bastion 0 ~]# yum install -y --releasever=8 --installroot=$scratchmnt redhat-release
~~~

You'll notice a few errors scroll
by but eventually you should see the following.

~~~shell
Installed:
  redhat-release-8.1-3.3.el8.x86_64                                           redhat-release-eula-8.1-3.3.el8.x86_64

Complete!
~~~

Next install the minimums to run shell programs. This will take a few minutes. 

~~~shell
[root@bastion 0 ~]# yum install -y --setopt=reposdir=/etc/yum.repos.d --installroot=$scratchmnt --setopt=cachedir=var/cache/dnf bash coreutils --setopt install_weak_deps=false 
~~~

Clean up the layers (confirm w/Dan)

~~~shell
[root@bastion 0 ~]# dnf clean --installroot $scratchmnt all
~~~

Again, you'll notice 
a few errors scroll by but eventually you should see the following.

~~~shell
Installed products updated.

Installed:
   coreutils-8.30-6.el8.x86_64 bash-4.4.19-10.el8.x86_64 libxkbcommon-0.8.2-1.el8.x86_64
   ...
   ...
   ...
   kmod-25-13.el8.x86_64 dbus-daemon-1:1.12.8-9.el8.x86_64 libgomp-8.3.1-4.5.el8.x86_64

Complete!
~~~

Create a simple shell script.

~~~shell
cat > runecho.sh <<EOF
#!/usr/bin/env bash
echo "Your simple container is working!"
EOF
~~~

Make the script executable and copy it to the `/usr/bin` directory
in the container's file system.

~~~shell
[root@bastion 0 ~]# chmod u+x runecho.sh
[root@bastion 0 ~]# buildah copy working-container ./runecho.sh /usr/bin

02a4078d6a03298c44d0962dea86e9b1a8f86e190af4270ea93caf75df36056f
~~~

Examine the container directory and you should see your shell script.

~~~shell
[root@bastion 0 ~]# ls -al $scratchmnt
~~~

Define an entry point. This command will run when the container runs.

~~~shell
[root@bastion 0 ~]# buildah config --entrypoint /usr/bin/runecho.sh working-container
~~~

Set some configuration labels.

~~~shell
[root@bastion 0 ~]# buildah config –author YourName –created-by buildah –label name=myshdemo working-container ~~~
~~~

Make a test run. Your echo script should run.

~~~shell
[root@bastion 0 ~]# buildah run --tty working-container /usr/bin/runecho.sh

Your simple container is working!
~~~

Make a change to `runecho.sh` and copy the file again.

~~~shell
[root@bastion 0 ~]# buildah copy working-container ./runecho.sh /usr/bin
~~~

Confirm the changes by running the container again.

~~~shell
[root@bastion 0 ~]# buildah run --tty working-container /usr/bin/runecho.sh
~~~

Commit the final verson to storage.

~~~shell
[root@bastion 0 ~]# buildah unmount working-container
[root@bastion 0 ~]# buildah commit working-container localhost/scratch

Getting image source signatures
Copying blob 3e01796ace31 done
Copying config 56fa90dd8f done
Writing manifest to image destination
Storing signatures
56fa90dd8fd9bf037d19b96f6990e698c99429518d1a747b25d8e98766f57c29
~~~

Exit the container.
~~~shell
[root@bastion 0 ~]# exit
~~~

Use `podman` to confirm the image was saved.

~~~shell
$ podman images
~~~

Test and run with `podman`.

~~~shell
$ podman run -it --rm localhost/scratch
$ podman  login
$ podman tag
$ podman push
~~~

Clean things up.

~~~shell
$ buildah ls
$ buildah rm working-container
$ podman rmi localhost/myapp:latest
~~~

#### Buildah CLI from UBI

~~~shell
$ buildah from --name=koz registry.access.redhat.com/ubi8/ubi
$ buildah run koz -- dnf -y install python3
$ echo "The container is working." > index.html
$ buildah copy koz index.html /
$ buildah config --cmd 'python3 -m http.server' koz
$ buildah config --author "wgh at redhat.com @twitter-handle" --label name=koz koz
$ buildah commit koz koz
$ podman run -d --name=test01 -p8000:8000 localhost/koz
~~~

#### Buildah using Docker (BuD)

Create the following `Dockerfile`

~~~shell
cat > Dockerfile <<EOF
FROM registry.access.redhat.com/ubi8/ubi
LABEL description="Minimal python web server" maintainer="yourname@mail.net"
RUN dnf -y update; dnf -y clean all
RUN dnf -y install python3 --setopt install_weak_deps=false; dnf -y clean all
RUN echo "The python http.server module is running." > /index.html
EXPOSE 8000
CMD [ "/usr/bin/python3",  "-m", "http.server" ]
EOF
~~~

Create a new container image from Dockerfile.

~~~shell
$ buildah bud -t buildahbuddemo .
~~~

List the images we have.

~~~shell
$ buildah images
~~~

Inspect the container image meta data

~~~shell
$ buildah inspect --type image buildahbuddemo
~~~

Maybe run some `skopeo` commands?
