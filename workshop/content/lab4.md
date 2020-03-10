## Registry Configuration
During this lab you will configure ```{{SERVER_0}}``` to host a secure 
container registry with authentication in a container.

#### Exercise: Registry Configuration

##### Overview 

* Configure the registry with with a self-signed certificate.
* Use curl to test connectivity to the registry services.

##### How to

Configure the registry on ```{{SERVER_0}}```.

~~~shell
$ cd $HOME/files/nodes/registry-files/gen-certs/
~~~

Edit ```myserver.cnf``` and confirm the ```FQDN``` is set to ```{{SERVER_0}}```.

Run the following commands to generate an SSL certificate.

~~~shell
$ sh ./gen-certs.sh
~~~

Now run the registry container.

~~~shell
$ cd ..
$ sh ./run-registry.sh
~~~

Test that the registry service is running.

~~~shell
$ curl  --user redhat:redhat https://{{SERVER_0}}:5000/v2/_catalog

{"repositories":[]}
~~~

Configure the registry on ```{{SERVER_1}}```.

~~~shell
$ cd $HOME/files/nodes/registry-files/gen-certs/
~~~

Edit ```myserver.cnf``` and confirm the ```FQDN``` is set to ```{{SERVER_1}}```.

Run the following commands to generate an SSL certificate.

~~~shell
$ sh ./gen-certs.sh
~~~

Now run the registry container.

~~~shell
$ cd ..
$ sh ./run-registry.sh
~~~

Test that the registry service is running.

~~~shell
$ curl  --user redhat:redhat https://{{SERVER_1}}:5000/v2/_catalog

{"repositories":[]}
~~~


Configure the clients on ```{{BASTION}}```.

First copy the SSL certificates from the registry servers and update the trust store.

~~~shell
$ sudo -i
# scp {{SERVER_0}}:/etc/pki/ca-trust/source/anchors/myserver.cert /etc/pki/ca-trust/source/anchors/myserver1.cert
# scp {{SERVER_1}}:/etc/pki/ca-trust/source/anchors/myserver.cert /etc/pki/ca-trust/source/anchors/myserver2.cert
# update-ca-trust
# exit
~~~

Now try to curl the registries from ```{{BASTION}}```.

~~~shell
$ curl  --user redhat:redhat https://{{SERVER_0}}:5000/v2/_catalog

{"repositories":[]}

$ curl  --user redhat:redhat https://{{SERVER_1}}:5000/v2/_catalog

{"repositories":[]}
~~~

From ```{{BASTION}}```, use ```curl``` to check basic networking to the registry.

~~~shell
$ curl http://{{SERVER_0}}/v2/_catalog
~~~

#### Exercise: Tagging and pushing images to a remote registry

##### Overview
* Loading a container image from an archive. 
* Tag and push an image to a remote registry.

Verify you have images to work with.

~~~shell
$ podman images
~~~

Expected Output:

~~~shell
REPOSITORY                                  TAG      IMAGE ID       CREATED         SIZE
registry.access.redhat.com/ubi8/python-36   latest   e3c71e2259e9   5 weeks ago     827 MB
quay.io/osevg/workshopper                   latest   e6fb0e1f52e2   13 months ago   847 MB
~~~

Now tag and push the image to the remote registry hosted at {{SERVER_0}}.

~~~shell
$ podman tag <IMAGE ID> {{SERVER_0}}:5000/ubi8/python-36
~~~

Confirm the tag is correct.

~~~shell
$ podman images
~~~

Expected Output:

~~~shell
REPOSITORY                             TAG      IMAGE ID       CREATED        SIZE
{{SERVER_0}}:5000/ubi8/python-36       latest   cc7efd763847   2 months ago   827 MB
~~~

Finally, login and push the image to the {{SERVER_0}} registry.

~~~shell
$ podman login
~~~

~~~shell
$ podman push {{SERVER_0}}:5000/ubi8/python-36
~~~

Expected Output:

~~~shell
Getting image source signatures
Copying blob 8da573feae5f: 205.77 MiB / 205.77 MiB [========================] 5s
Copying blob 6ef321d2357f: 10.00 KiB / 10.00 KiB [==========================] 5s
Copying config cc7efd763847: 0 B / 4.36 KiB [-------------------------------] 0s
Writing manifest to image destination
Writing manifest to image destination
Storing signatures
~~~

#### Exercise: Blocking a remote registry

##### Overview

TODO: Move this to image signing lab. 

Blocked Registries is deprecated because other container runtimes and tools will not use it.
It is recommended that you use the trust policy file /etc/containers/policy.json to control which
registries you want to allow users to pull and push from.  policy.json gives greater flexibility, and
supports all container runtimes and tools including the docker daemon, cri-o, buildah ...

Remove the local copy.

~~~shell
$ podman rmi {{SERVER_1}}:5000/ubi8/python-36
~~~

Now try to pull the image from {{SERVER_1}}, it should fail.

~~~shell
$ podman pull {{SERVER_1}}:5000/ubi8/python-36
~~~