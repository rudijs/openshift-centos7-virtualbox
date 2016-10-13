# openshift-origin-docker-osx
Run OpenShift Origin on Osx using Virtualbox and Docker

## Requirements

- [Docker Toolbox](https://www.docker.com/products/docker-toolbox)
- [Virtualbox](https://www.virtualbox.org/)
- [Vagrant](https://www.vagrantup.com/)

## Destroy

Before we begin a new install, you may need to destroy an existing install.

If so use these three steps in one command:

1. Remove the virtual machine (VM)
2. Remove from ./ssh/known_hosts
3. Remove from docker-machine

Run this command:

- `vagrant destroy; ssh-keygen -R 192.168.33.10; docker-machine rm openshift-origin`  

## SSH Key

Lets setup passwordless SSH login for the virtual machine we'll use.

Your ssh private key **should** be password protected, to use it passwordless add it to the ssh authentication agent

Run this command:

- `ssh-add` (enter your password).

## Install

Run these commands:

1) Clone the git project into the encrypted directory

- `git clone git@github.com:rudijs/openshift-origin-docker-osx.git`

2) Start the Virtual Machine (VM)

- `cd openshift-origin-docker-osx`
- `vagrant up`

3) Copy your public ssh key for passwordless login (`brew install ssh-copy-id` if you need it on OSX)

- `ssh-copy-id vagrant@192.168.33.10` (password is 'vagrant')

**Important**: Test you are able to connect *without password* to your VM

- `ssh -v vagrant@192.168.33.10`

If you are able to log in without password you're good to proceed.

4) Install docker into the VM

- `docker-machine create --driver generic --generic-ssh-user vagrant --generic-ip-address 192.168.33.10 openshift-origin`

This is a sample of the output you should see:

```
$ docker-machine create --driver generic --generic-ssh-user vagrant --generic-ip-address 192.168.33.10 openshift-origin
Running pre-create checks...
Creating machine...
(openshift-origin) No SSH key specified. Connecting to this machine now and in the future will require the ssh agent to contain the appropriate key.
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with ubuntu(upstart)...
Installing Docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env openshift-origin
```

Next lets test out our docker client and server.

5) Point your workstation docker client to the VM

- `eval "$(docker-machine env openshift-origin)"`

6) Review your docker client

- `docker-machine ls`

7)
The install of docker will have updated our server and most likely upgraded to a new kernel, lets reboot into the new to be sure:

- `vagrant reload`

8) Lets add NTP to keep our server's time in sync

- `vagrant ssh -c 'sudo apt-get install ntp -yy'`

9)

Finally, now it's time for OpenShift Origin.

Run this docker command:

```
docker run -d --name "origin" \
        --privileged --pid=host --net=host \
        -v /:/rootfs:ro -v /var/run:/var/run:rw -v /sys:/sys -v /var/lib/docker:/var/lib/docker:rw \
        -v /var/lib/origin/openshift.local.volumes:/var/lib/origin/openshift.local.volumes \
        --net host openshift/origin:latest start --public-master=https://192.168.33.10:8443
```

Open your browser at the web admin console

- [https://192.168.33.10:8443/console](https://192.168.33.10:8443/console)

You'll see a web browser warning about the untrusted self-signed TLS certificate, it's OK to accept that.

Login with the credentials:

- Username: `test`
- Password: `test`

*Note*: If you want or need a specific version of OpenShift Origin you can pin the version by replacing `latest` with a version (example: *v1.2.1)

See the [available tagged versions here](https://hub.docker.com/r/openshift/origin/tags/)
