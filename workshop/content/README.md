# DRAFT: A Practical Introduction to Container Security

This lab guide is written in markdown and is intended to be rendered by the [workshopper](https://github.com/openshift-evangelists/workshopper) software. However, you do not have to clone this repo to host the lab guide. 
A Linux container engine like ```podman``` or ```docker``` can be used to host it. 

For example, to host the lab guide on your local RHEL or Fedora system, run the following command.

```
$ podman run -d -p 8080:8080 --name=lab-guide -e CONTENT_URL_PREFIX="https://gitlab.com/2020-summit-labs/a-practical-introduction-to-container-security/raw/master/" -e WORKSHOPS_URLS="https://gitlab.com/2020-summit-labs/a-practical-introduction-to-container-security/raw/master/_workshop1.yml" quay.io/osevg/workshopper
```

Then visit [http://localhost:8080](http://localhost:8080) to access the lab guide.

##### RHPDS users can host the lab guide from the remote client system.

1) Once the lab is provisioned and you have a GUID, use ```ssh``` to login to the client system.

~~~shell
$ ssh rhpds-user-name@client-GUID.rhpds.opentlc.com
~~~

2) Run the ```podman``` command from above.

3) Visit lab guide at [http://client-GUID.rhpds.opentlc.com:8080](client-GUID.rhpds.opentlc.com:8080).

##### DIY
To build out your own lab, just create (2) RHEL 8 machines. The ```scripts``` directory will help you install the Quay registry on the ```server.summit.lab``` system.
