---
title: Testing Openshift Origin V3 with Ansible and Vagrant on OS X
layout: post
tags:
 - ansible
 - mac
 - openshift
 - vagrant
---

The [OpenShift Origin](http://www.openshift.org/) project provides [Ansible](http://www.ansible.com) playbooks and roles for installing OpenShift on various infratructure. I'm going to try out the example using [Vagrant](http://www.vagrantup.com) and [VirtualBox](https://www.virtualbox.org/) on my Mac. I'm not very familiar with Vagrant or OpenShift v3 yet, so I'm just going to think out loud and see how it goes.

## Some Background ##

OpenShift Origin is an opensource PaaS (platform as a service). It is the upstream project for Red Hat's [OpenShift Online](https://www.openshift.com/) and [OpenShift Enterprise](https://enterprise.openshift.com/). Version 3 of the OpenShift platform is a complete rewrite _just_ launched in June 2015. It now utilizes [Docker](http://www.docker.com) as the container engine and [Kubernetes](http://kubernetes.io/) as the orchestrator. The Enterprise edition uses [Red Hat Atomic Enterprise Platform](https://access.redhat.com/products/red-hat-atomic-enterprise-platform) as the underlying OS. The example used in this post will create Vagrant CentOS boxes.

## Initial Setup ##

- First you'll need to install [Vagrant](https://www.vagrantup.com/) on your Mac.

- Next clone the [openshift-ansible repo](https://github.com/openshift/openshift-ansible). 

- Now following the instructions in [README_vagrant.md](https://github.com/openshift/openshift-ansible/blob/master/README_vagrant.md) create three boxes that will form the OpenShift cluster. One master and two nodes.

{% highlight bash %}
cd ~/src && git@github.com:openshift/openshift-ansible.git
cd openshift-ansible
# Install the requisite vagrant plugins
$ vagrant plugin install vagrant-hostmaster
# Create the virtual boxes, but don't run ansible yet
$ vagrant up --no-provision

# Later we will provision like this:
#  vagrant provision
# When we want to try again it will look like this
#  vagrant reload --provision
{% endhighlight %}

There are now 3 machines (boxes) which where added to `/etc/hosts` by vagrant.

{% highlight text %}
tail -5 /etc/hosts
## vagrant-hostmanager-start id: bfad3436-5a92-4f03-b555-55bd186dd0ba
192.168.100.200	ose3-node1.example.com
192.168.100.201	ose3-node2.example.com
192.168.100.100	ose3-master.example.com
## vagrant-hostmanager-end
{% endhighlight %}

You can see they are running

![virtual box screenshot](/images/openshift-virtualbox-0.png)

{% highlight bash %}
$ vagrant status
Current machine states:

node1                     running (virtualbox)
node2                     running (virtualbox)
master                    running (virtualbox)
{% endhighlight %}

They can be accessed over ssh using their short vagrant names like this:

{% highlight bash %}
$ vagrant ssh master
$ vagrant ssh node1
$ vagrant ssh node2
{% endhighlight %}

Not only that, but port 8443 on the Mac localhost is forwarded to port 8443 on the master node. Nothing is listening on the master just yet though.

On to the provisioning step.

## Provisioning with Ansible ##

**WARNING** Before you continue, check if these issues are closed yet. If not, apply the patches referenced by them.

- [Issue 331](https://github.com/openshift/openshift-ansible/issues/331)
- [Issue 336](https://github.com/openshift/openshift-ansible/issues/336)

Run the [byo/config.yml](https://github.com/openshift/openshift-ansible/blob/master/playbooks/byo/config.yml) Ansible playbook on the cluster by way of the vagrant provision command

{% highlight bash %}
$ vagrant provision
{% endhighlight %}
## Sanity Check OpenShift ##

Expect an _ok_ from the healthcheck

{% highlight bash %}
$ vagrant ssh node2 
[vagrant@ose3-node2 ~]$ curl -k https://ose3-master.example.com:8443/healthz
ok
{% endhighlight %}

The OpenShift console can be reached at https://127.0.0.1:8443/console but how can I reach https://ose3-master.example.com:8443/console ?

For now I just added the name to my localhost line in `/etc/hosts`, but is there a better way to do this automatically with Vagrant?

{% highlight text %}
$ grep localhost /etc/hosts
# localhost is used to configure the loopback interface
127.0.0.1 localhost ose3-master.example.com
::1             localhost
{% endhighlight %}

![console screenshot](/images/openshift-console-0.png)

OpenShift console command `oc` is similar to `kubectl`. Let's blindly try a few commands.

{% highlight bash %}
$ vagrant ssh master

[vagrant@ose3-master ~]$ oc get nodes
NAME                     LABELS                                          STATUS
ose3-node1.example.com   kubernetes.io/hostname=ose3-node1.example.com   Ready
ose3-node2.example.com   kubernetes.io/hostname=ose3-node2.example.com   Ready

[vagrant@ose3-master ~]$ oc get services
NAME            LABELS                                    SELECTOR   IP(S)        PORT(S)
kubernetes      component=apiserver,provider=kubernetes   <none>     172.30.0.2   443/TCP
kubernetes-ro   component=apiserver,provider=kubernetes   <none>     172.30.0.1   80/TCP

[vagrant@ose3-master ~]$ oc get replicationcontrollers
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR   REPLICAS

[vagrant@ose3-master ~]$ oc get all
NAME      TYPE      SOURCE
NAME      TYPE      STATUS    POD
NAME      DOCKER REPO   TAGS      UPDATED
NAME      TRIGGERS   LATEST VERSION
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR   REPLICAS
NAME      HOST/PORT   PATH      SERVICE   LABELS
NAME            LABELS                                    SELECTOR   IP(S)        PORT(S)
kubernetes      component=apiserver,provider=kubernetes   <none>     172.30.0.2   443/TCP
kubernetes-ro   component=apiserver,provider=kubernetes   <none>     172.30.0.1   80/TCP
NAME      READY     REASON    RESTARTS   AGE
{% endhighlight %}

## Configure OpenShift ##

_I haven't finished going through this bit_

Now to walk through the OpenShift [getting started](https://github.com/openshift/origin#getting-started) docs.

### Create Docker Registry ###

**BUG** _this fails_

- Create a docker registry.

{% highlight bash %}
$ vagrant ssh master
[vagrant@ose3-master ~]$ export CURL_CA_BUNDLE=/etc/openshift/master/ca.crt
[vagrant@ose3-master ~]$ export KUBECONFIG=/etc/openshift/master/admin.kubeconfig
[vagrant@ose3-master ~]$ sudo chmod +r /etc/openshift/master/openshift-registry.kubeconfig
[vagrant@ose3-master ~]$ sudo chmod +r $KUBECONFIG

[vagrant@ose3-master ~]$ oadm registry --create --credentials=/etc/openshift/master/openshift-registry.kubeconfig --config=$KUBECONFIG
deploymentconfigs/docker-registry
services/docker-registry

[vagrant@ose3-master ~]$ oc get pods
NAME                       READY     REASON    RESTARTS   AGE
docker-registry-1-deploy   0/1       Pending   0          10s

[vagrant@ose3-master ~]$ oc get pods
NAME                       READY     REASON         RESTARTS   AGE
docker-registry-1-deploy   0/1       ExitCode:255   0          2m

[vagrant@ose3-master ~]$ oc get all
NAME      TYPE      SOURCE
NAME      TYPE      STATUS    POD
NAME      DOCKER REPO   TAGS      UPDATED
NAME              TRIGGERS       LATEST VERSION
docker-registry   ConfigChange   1
CONTROLLER          CONTAINER(S)   IMAGE(S)                                  SELECTOR                                                                                REPLICAS
docker-registry-1   registry       openshift/origin-docker-registry:v1.0.0   deployment=docker-registry-1,deploymentconfig=docker-registry,docker-registry=default   0
NAME      HOST/PORT   PATH      SERVICE   LABELS
NAME              LABELS                                    SELECTOR                  IP(S)           PORT(S)
docker-registry   docker-registry=default                   docker-registry=default   172.30.70.190   5000/TCP
kubernetes        component=apiserver,provider=kubernetes   <none>                    172.30.0.2      443/TCP
kubernetes-ro     component=apiserver,provider=kubernetes   <none>                    172.30.0.1      80/TCP
NAME                       READY     REASON         RESTARTS   AGE
docker-registry-1-deploy   0/1       ExitCode:255   0          2m
{% endhighlight %}


### Create OpenShift App ###

**BUG** _this fails_

- Login as _test_ / _test_ then create a project and an app. This will peform a docker build, but will fail when it attempts to push to the registry above.

{% highlight text %}
$ vagrant ssh master
[vagrant@ose3-master ~]$ oc login
Username: test
Authentication required for https://ose3-master.example.com:8443 (openshift)
Password:
Login successful.

You don't have any projects. You can try to create a new project, by running

    $ oc new-project <projectname>

[vagrant@ose3-master ~]$ oc new-project test
Now using project "test" on server "https://ose3-master.example.com:8443".

[vagrant@ose3-master ~]$ oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/sample-app/application-template-stibuild.json
services/frontend
routes/route-edge
imagestreams/origin-ruby-sample
imagestreams/ruby-20-centos7
buildconfigs/ruby-sample-build
deploymentconfigs/frontend
services/database
deploymentconfigs/database
Service "frontend" created at 172.30.228.57 with port mappings 5432->8080.
A build was created - you can run `oc start-build ruby-sample-build` to start it.
Service "database" created at 172.30.167.96 with port mappings 5434->3306.

[vagrant@ose3-master ~]$ oc status
In project test

service database (172.30.167.96:5434 -> 3306)
  database deploys docker.io/openshift/mysql-55-centos7:latest
    #1 deployment pending 25 seconds ago

service frontend (172.30.228.57:5432 -> 8080)
  frontend deploys origin-ruby-sample:latest <-
    builds https://github.com/openshift/ruby-hello-world.git with test/ruby-20-centos7:latest
    build 1 running for 20 seconds
    #1 deployment waiting on image or update

To see more information about a Service or DeploymentConfig, use 'oc describe service <name>' or 'oc describe dc <name>'.
You can use 'oc get all' to see lists of each of the types described above.

[vagrant@ose3-master ~]$ oc get pods
NAME                        READY     REASON         RESTARTS   AGE
database-1-deploy           0/1       ExitCode:255   0          1m
ruby-sample-build-1-build   0/1       ExitCode:255   0          1m
{% endhighlight %}

### Check out the OpenShift Console ###

- Add admin user to the _test_ Project.

{% highlight bash %}
$ vagrant ssh master
[vagrant@ose3-master ~]$ oadm policy add-role-to-user admin admin -n test
{% endhighlight %}

Be sure you updated your hosts file as described above then browse to one of the following and login as _admin_ / _admin_: 

- https://localhost:8443/console/
- https://ose3-master.example.com:8443/console/

Once you click into the _test_ project you'll see broken services like this.

![openshift console test screenshot](/images/openshift-console-test-0.png)
