---
title: Testing Openshift Origin V3 with Ansible and Vagrant on OS X
layout: post
tags:
 - ansible
 - mac
 - openshift
 - vagrant
---

The [OpenShift Origin](http://www.openshift.org/) project provides [Ansible](http://www.ansible.com) playbooks and roles for installing OpenShift on various infratructure. I'm going to try out the example using [Vagrant](http://www.vagrantup.com) and [VirtualBox](https://www.virtualbox.org/) on my Mac. I'm not very familiar with Vagrant of OpenShift v3 yet, so I'm just going to think out loud and see how it goes.

## Some Background ##

OpenShift Origin is an opensource platform as a service. It is the upstream project for Red Hat's [OpenShift Online](https://www.openshift.com/) and [OpenShift Enterprise](https://enterprise.openshift.com/). Version 3 of the OpenShift platform is a complete rewrite _just_ launched in June 2015. It now utilizes [Docker](http://www.docker.com) as the container engine and [Kubernetes](http://kubernetes.io/) as the orchestrator. The Enterprise edition uses [Red Hat Atomic Enterprise Platform](https://access.redhat.com/products/red-hat-atomic-enterprise-platform) as the underlying OS. The example used in this post will create Vagrant CentOS boxes.

## Initial Setup ##

- First you'll need to install [Vagrant](https://www.vagrantup.com/) on your Mac.

- Next clone the [openshift-ansible repo](https://github.com/openshift/openshift-ansible). 

- Now following the instructions in [README_vagrant.md](https://github.com/openshift/openshift-ansible/blob/master/README_vagrant.md) create three boxes that will form the OpenShift cluster. One master and two nodes.

{% highlight bash %}
cd ~/src && git@github.com:openshift/openshift-ansible.git
cd openshift-ansible
# Download the virtual boxes, but don't run ansible yet
vagrant plugin install vagrant-hostmaster
vagrant up --no-provision
# Later we will provision like this:
# vagrant provision
# When we want to try again it will look like this
# vagrant reload --provision
{% endhighlight %}

There are now 3 machines (boxes) which where added to `/etc/hosts` by vagrant.

{% highlight text %}
## vagrant-hostmanager-start id: bfad3436-5a92-4f03-b555-55bd186dd0ba
192.168.100.200	ose3-node1.example.com
192.168.100.201	ose3-node2.example.com
192.168.100.100	ose3-master.example.com
## vagrant-hostmanager-end
{% endhighlight %}

They are running

{% highlight bash %}
vagrant status
Current machine states:

node1                     running (virtualbox)
node2                     running (virtualbox)
master                    running (virtualbox)
{% endhighlight %}

They can be accessed over ssh I can ssh to them using their short vagrant names like this

{% highlight bash %}
vagrant ssh master
vagrant ssh node1
vagrant ssh node2
{% endhighlight %}

Not only that, port 8443 on the Mac is forwarded to port 8443 on the master node. (_Hold that thought! There's a problem with redirects and FQDN later._)

On to the provisioning step.

**NOTE** Before you continue! I [finally realized](https://github.com/openshift/openshift-ansible/pull/319#issuecomment-119374507) that at this point NetworkManager is erroneously taking over responsiblity for the `enp0s8` interface, and the `192.168.100.x` addresses will disappear in short order. Subsequently the playbook will fail. I'm not sure the best way to fix this. The workaround is to simply reboot the nodes at this point. Otherwise you can do this for all three boxes.

{% highlight bash %}
vagrant ssh master
sudo systemctl restart NetworkManager
sudo systemctl restart network
{% endhighlight %}

## Provisioning with Ansible ##

- Run the [byo/config.yml](https://github.com/openshift/openshift-ansible/blob/master/playbooks/byo/config.yml) Ansible playbook on the cluster by way of the vagrant provision command

{% highlight bash %}
vagrant provision
{% endhighlight %}

### Bug! Playbook Hangs on openshift_manage_node role ###


{% highlight text %}
...
PLAY [Set scheduleability] ****************************************************

GATHERING FACTS ***************************************************************
ok: [master]

TASK: [set_fact ] *************************************************************
ok: [master]

TASK: [openshift_manage_node | Handle unscheduleable node] ********************
skipping: [master]

TASK: [openshift_manage_node | Handle scheduleable node] **********************
failed: [master] => (item=ose3-node1.example.com) => {"changed": true, "cmd": ["oadm", "manage-node", "ose3-node1.example.com", "--schedulable=true"], "delta": "0:00:00.359376", "end": "2015-07-07 23:28:36.050071", "item": "ose3-node1.example.com", "rc": 1, "start": "2015-07-07 23:28:35.690695", "warnings": []}
stderr: error: No nodes found
failed: [master] => (item=ose3-node2.example.com) => {"changed": true, "cmd": ["oadm", "manage-node", "ose3-node2.example.com", "--schedulable=true"], "delta": "0:00:00.354480", "end": "2015-07-07 23:28:36.698472", "item": "ose3-node2.example.com", "rc": 1, "start": "2015-07-07 23:28:36.343992", "warnings": []}
stderr: error: No nodes found

FATAL: all hosts have already failed -- aborting

PLAY RECAP ********************************************************************
           to retry, use: --limit @/Users/dlbewley/config.retry

localhost                  : ok=5    changed=0    unreachable=0    failed=0
master                     : ok=59   changed=27   unreachable=0    failed=1
node1                      : ok=46   changed=20   unreachable=0    failed=0
node2                      : ok=46   changed=20   unreachable=0    failed=0
{% endhighlight %}

Even though this works:

{% highlight text %}
[vagrant@ose3-master ~]$ oc get nodes
NAME                     LABELS                                          STATUS
ose3-node1.example.com   kubernetes.io/hostname=ose3-node1.example.com   Ready
ose3-node2.example.com   kubernetes.io/hostname=ose3-node2.example.com   Ready
[vagrant@ose3-master ~]$ oadm manage-node ose3-node2.example.com --schedulable=true
NAME                     LABELS                                          STATUS
ose3-node2.example.com   kubernetes.io/hostname=ose3-node2.example.com   Ready
[vagrant@ose3-master ~]$ oadm manage-node ose3-node1.example.com --schedulable=true
NAME                     LABELS                                          STATUS
ose3-node1.example.com   kubernetes.io/hostname=ose3-node1.example.com   Ready
{% endhighlight %}

That is in [this task file](https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_manage_node/tasks/main.yml).

Try running `vagrant provision` again and it succeeds. I need to complete a fresh test.

{% highlight text %}
PLAY [Set scheduleability] ****************************************************

GATHERING FACTS ***************************************************************
ok: [master]

TASK: [set_fact ] *************************************************************
ok: [master]

TASK: [openshift_manage_node | Handle unscheduleable node] ********************
skipping: [master]

TASK: [openshift_manage_node | Handle scheduleable node] **********************
changed: [master] => (item=ose3-node1.example.com)
changed: [master] => (item=ose3-node2.example.com)

PLAY RECAP ********************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0
master                     : ok=52   changed=1    unreachable=0    failed=0
node1                      : ok=41   changed=0    unreachable=0    failed=0
node2                      : ok=41   changed=0    unreachable=0    failed=0
{% endhighlight %}


## Sanity Check OpenShift ##

Expect an _ok_ from the healthcheck

{% highlight bash %}
vagrant ssh node2 
[vagrant@ose3-node2 ~]$ curl -k https://ose3-master.example.com:8443/healthz
ok
{% endhighlight %}

The OpenShift console can be reached at https://127.0.0.1:8443/console but how can I reach https://ose3-master.example.com:8443/console ?

For now I just added the name to my localhost line in `/etc/hosts`, but is there a better way to do this automatically with Vagrant?

{% highlight text %}
127.0.0.1	localhost ose3-master.example.com
{% endhighlight %}

![console screenshot](/images/openshift-console-0.png)

## Configure OpenShift ##

_I haven't finished going through this bit_

Off to see [getting started](https://github.com/openshift/origin#getting-started). 

- Create a docker registry. _this fails_

{% highlight bash %}
vagrant ssh master
[vagrant@ose3-master ~]$ oadm registry --credentials=./openshift.local.config/master/openshift-registry.kubeconfig
[vagrant@ose3-master ~]$ oc get pods
NAME                       READY     REASON    RESTARTS   AGE
docker-registry-1-deploy   0/1       Pending   0          47s
{% endhighlight %}

- Login as test / test then create a project and an app. This will peform a docker build, but will fail when it attempts to push to the registry above.

{% highlight text %}
vagrant ssh master
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
Service "frontend" created at 172.30.184.9 with port mappings 5432->8080.
A build was created - you can run `oc start-build ruby-sample-build` to start it.
Service "database" created at 172.30.219.1 with port mappings 5434->3306.

[vagrant@ose3-master ~]$ oc status
In project test

service database (172.30.219.1:5434 -> 3306)
  database deploys docker.io/openshift/mysql-55-centos7:latest
    #1 deployment pending 50 seconds ago

service frontend (172.30.184.9:5432 -> 8080)
  frontend deploys origin-ruby-sample:latest <-
    builds https://github.com/openshift/ruby-hello-world.git with test/ruby-20-centos7:latest
    build 1 pending for 38 seconds
    #1 deployment waiting on image or update

To see more information about a Service or DeploymentConfig, use 'oc describe service <name>' or 'oc describe dc <name>'.
You can use 'oc get all' to see lists of each of the types described above.			
{% endhighlight %}

Be sure to check out the console and login as test / test

- https://localhost:8443/console/
- https://ose3-msater.example.com:8443/console/
