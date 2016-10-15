# openshift-centos7-virtualbox

Centos 7 Virtual Machine running OpenShift

## Overview

This project aims to enable a local developer version which matches the technology stack of production system.

As the title says, it's a Centos7 virtual machine running and Openshift binary.

In production there are multiple machines, master and nodes, but this local stack will be all-in-one on one VM.

## Install

The installation process is simply running commands in a local shell and also commands inside the running VM.

- Clone the project repository
- `git clone https://github.com/rudijs/openshift-centos7-virtualbox.git`
- `cd openshift-centos7-virtualbox`
- `vagrant up`
- SSH into the VM
- `vagrant ssh`
- Check that Selinux is enabled
- `sudo vi /etc/sysconfig/selinux`
- If not set to **enforcing** update to `SELINUX=enforcing` and reboot
- Install Docker. We'll use the curl bash method. You can reivew [here](https://docs.docker.com/engine/installation/linux/centos/) is you wish
- `sudo yum update -y`
- `curl -fsSL https://get.docker.com/ | sh`
- `sudo systemctl enable docker.service`
- `sudo systemctl start docker`
- From the Openshift releases download the version of openshift you wish to run [https://github.com/openshift/origin/releases](https://github.com/openshift/origin/releases)
- In this case we'll download v1.3.0
- `curl -L https://github.com/openshift/origin/releases/download/v1.3.0/openshift-origin-server-v1.3.0-3ab7af3d097b57f933eccef684a714f2368804e7-linux-64bit.tar.gz -o openshift-origin-server-v1.3.0-3ab7af3d097b57f933eccef684a714f2368804e7-linux-64bit.tar.gz`
- Extract the openshift archive into the /opt directory
- `sudo tar -C /opt -zxvf openshift-origin-server-v1.3.0-3ab7af3d097b57f933eccef684a714f2368804e7-linux-64bit.tar.gz` 
- `sudo ln -s /opt/openshift-origin-server-v1.3.0-3ab7af3d097b57f933eccef684a714f2368804e7-linux-64bit/ /opt/openshift`
- Lets now start openshift from the command line and login with the web browser
- `sudo /opt/openshift/openshift start --public-master=https://192.168.33.10:8443`
- You'll see lots of logging fly by on the screen.
- Open up a browser at [https://192.168.33.10:8443](https://192.168.33.10:8443)
- Login with the user/password **admin** **admin**

<!--

- Now is a good time to setup start on boot for Openshift.
- Exit on the command line with `clt-c`
- Create and openshift systemd service file
- `sudo vi openshift.service`
- Paste this text into that file:

[Unit]
Description=Origin Service
Documentation=https://github.com/openshift/origin
After=network.target
After=docker.service

[Service]
Type=notify
ExecStart=/opt/openshift/openshift start --public-master=https://192.168.33.10:8443
LimitNOFILE=131072
LimitCORE=infinity
SyslogIdentifier=origin-master
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target

- `sudo systemctl enable openshift.service`
- `sudo systemctl start openshift`
- After a few moments openshift will be started, check for the process with:
- `ps aux | grep openshift`
- Open up a browser at [https://192.168.33.10:8443](https://192.168.33.10:8443)

-->

- Next up we'll run some configurations and start some services we need.
- Your current admin account doesn't have full admin privileges, lets update that.
- `cd /opt/openshift`
- login as system:admin
- `sudo ./oc login -u system:admin -n default --config=openshift.local.config/master/admin.kubeconfig`
- Add cluster admin privileges to user 'admin'
- `sudo ./oadm policy add-cluster-role-to-user cluster-admin admin  --config=openshift.local.config/master/admin.kubeconfig`
- In your web browser logged in as the user admin, you'll now see admin privileged view of system projects.
- Next up lets start an integrated docker registry service for our project to use.
- `sudo ./oadm registry --config=openshift.local.config/master/admin.kubeconfig --service-account=registry`
- In your web browser, click to check the default project. You'll see the docker-registry container starting up.
- Click on **Applications** and **Builds** to view the status of the docker-registry.
- Before our projects can use this integrated docker registry we need to update the system docker process.
- We need the IP of the docker-registry service, click **Applications** then **Services**
- Copy the **Cluster IP** of the docker-registry. For me it's **172.30.201.17**
- Edit the system docker config and restart the docker service.
- `sudo vi /lib/systemd/system/docker.service`
- Edit the **ExecStart** line with your docker-registry IP
- `ExecStart=/usr/bin/dockerd --selinux-enabled --insecure-registry 172.30.201.17:5000`
- `sudo systemctl daemon-reload`
- `sudo systemctl restart docker`
- Next up we'll add a router for our projects to use.
- For the next few steps we'll switch to the root user
- `sudo su -`
- We'll export the admin.kubeconfig so we don't have to pass it in as we've been doing with the --config option
- `export KUBECONFIG=/opt/openshift/openshift.local.config/master/admin.kubeconfig`
- `cd /opt/openshift`
- `./oadm policy add-scc-to-user hostnetwork -z router`
- `./oadm router --service-account=router`
- `./oadm policy reconcile-cluster-roles`
- If you want or need to remove or re-add the router you can remove it with `./oc delete all -l router=router`
- You should now be able to see the router container running in the default project along with the docker-registry container.
- Exit from root account back to user vagrant with `ctl-c`
- OK, now we're ready to start to build and launch our own custom projects.
- Lets install git and clone a simple nodejs http server project.
- `sudo yum -y install git`
- From the command line login as the regular admin user account
- `./oc login -u admin`
- Create a new project named **app**
- `./oc new-project app`
- Create a new app in the **app** project
- `./oc new-app https://github.com/rudijs/os-httpd.git`
- Open your web browser, click into the **app** project, under **Builds** and **Applications** you'll see the app build and deploy.
- Lets now create a public route to our new app.
- Click **Applications** then **Services** then **os-httpd**
- Click Routes: **Create route** then **Create**
- Set your /etc/hosts or equivilent in Windows to have the host entry
- `192.168.33.100 os-httpd-app.router.default.svc.cluster.local`
- Click or open [http://os-httpd-app.router.default.svc.cluster.local/](http://os-httpd-app.router.default.svc.cluster.local/) in your browser.
- You should see 'Hello World'
- All done! Congratulations :-)
- To completely remove the app you can use `./oc delete all -l app=os-httpd`

<!--
https://github.com/anacortes/openshift-demo/blob/master/README.md
https://docs.docker.com/engine/installation/linux/centos/
-->
