## Introduction

This lab session is a low-level, hands-on introduction to container security using the Red Hat Enterprise Linux 8 command line interface. It is intended to be consumed as a series of self paced exercises.

### Prerequisites

* An introductory knowledge of Linux and containers is helpful.
* Basic text editing skills using vim or nano.

#### Lab Environment

![Lab Diagram]({% image_path lab-diagram.png %})

#### System Information

You will be working with the following systems running Red Hat Enterprise Linux 8 beta. 

* {{BASTION}} (Container Engine and ssh access)
* {{SERVER_0}} (Container Registry)

##### Lab Guide

The lab guide [source](https://gitlab.com/2020-summit-labs/a-practical-introduction-to-container-security) is available.
The [README.md](https://gitlab.com/2020-summit-labs/a-practical-introduction-to-container-security/blob/master/README.md) contains instructions for rendering and hosting the lab guide. 
All you need is a couple of RHEL8 servers to run this lab yourself.

##### Conventions used in this lab 

Any example that begins with the ```$``` prompt is meant to be executed on the container host.

For example:

~~~shell
$ ls /usr/tmp
~~~

or

~~~shell
$ sudo ls /var/lib/containers
~~~

Any example that begins with the ```{{CONTAINER_PROMPT}}``` prompt is meant to be executed in the container's namespace.

For example:

~~~shell
{{CONTAINER_PROMPT}} ls /
~~~

Commands are shown in fixed width font followed by a
blank line along with **sample** output.

For example:

~~~shell
$ podman ps -a

CONTAINER ID  IMAGE                       COMMAND    CREATED      STATUS   PORTS  NAMES
5d6230fd6fdd  localhost/rhel-ubi8:latest  sleep 999  9 hours ago  Created         sleepy
~~~

#### Getting Started Exercise

##### Login to the systems.

1) Client.

~~~shell
$ ssh rhpds-user-name@client-GUID.rhpds.opentlc.com
~~~

2) Server.

~~~shell
$ ssh rhpds-user-name@server-GUID.rhpds.opentlc.com
~~~

3) Confirm the following command does not prompt for a password and does not report any errors on **both** client and server systems.

~~~shell
$ sudo ls /var/lib/containers/

cache  storage
~~~

Now its time to dig in.

**=>**  Make a **backup** copy before modifying any file on the systems.  
