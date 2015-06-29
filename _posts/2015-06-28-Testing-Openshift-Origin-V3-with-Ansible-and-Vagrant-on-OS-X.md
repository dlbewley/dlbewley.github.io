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

## Provisioning with Ansible ##

- Run the [byo/config.yml](https://github.com/openshift/openshift-ansible/blob/master/playbooks/byo/config.yml) Ansible playbook on the cluster by way of the vagrant provision command

{% highlight bash %}
vagrant provision
{% endhighlight %}

### Bug! Playbook Hangs on openshift_node role ###

I found that [this task file](https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_node/tasks/main.yml) will eventually hang on the _Check scheduleable state_ task for some reason.

{% highlight text %}
TASK: [openshift_node | Start and enable openshift-node] **********************
changed: [node2]
changed: [node1]

TASK: [openshift_node | Check scheduleable state] *****************************
failed: [node1 -> master] => {"attempts": 10, "changed": true, "cmd": ["oc", "get", "node", "ose3-node1.example.com"], "delta": "0:00:00.237067", "end": "2015-06-29 04:40:36.806024", "failed": true, "rc": 1, "start": "2015-06-29 04:40:36.568957", "warnings": []}
stderr: Error from server: minion "ose3-node1.example.com" not found
msg: Task failed as maximum retries was encountered
failed: [node2 -> master] => {"attempts": 10, "changed": true, "cmd": ["oc", "get", "node", "ose3-node2.example.com"], "delta": "0:00:00.234398", "end": "2015-06-29 04:40:38.116159", "failed": true, "rc": 1, "start": "2015-06-29 04:40:37.881761", "warnings": []}
stderr: Error from server: minion "ose3-node2.example.com" not found
msg: Task failed as maximum retries was encountered

FATAL: all hosts have already failed -- aborting
{% endhighlight %}

The task looks like this:

{% highlight yaml %}
- name: Check scheduleable state
  delegate_to: "{{ openshift_first_master }}"
  command: >
    {{ openshift.common.client_binary }} get node {{ openshift.common.hostname }}
  register: ond_get_node
  until: ond_get_node.rc == 0
  retries: 10
  delay: 5
{% endhighlight %}

Logging into the master and trying the command does fail.

{% highlight bash %}
$ vagrant ssh master
[vagrant@ose3-master ~]$ oc get node ose3-node1.example.com
Error from server: minion "ose3-node1.example.com" not found
[vagrant@ose3-master ~]$ oc get node ose3-node2.example.com
Error from server: minion "ose3-node2.example.com" not found
[vagrant@ose3-master ~]$ oc get nodes
NAME      LABELS    STATUS
{% endhighlight %}

Try running `vagrant provision` again and it seems to hang indefinitely until hitting `^C`.

{% highlight text %}
TASK: [openshift_node | Start and enable openshift-node] **********************
ok: [node2]
ok: [node1]

TASK: [openshift_node | Check scheduleable state] *****************************
{% endhighlight %}


I'm not sure why it hangs, because at the same time I can run that same command without a hang.

{% highlight bash %}
mac$ vagrant ssh master
[vagrant@ose3-master log]$ oc get node
NAME                     LABELS                                          STATUS
ose3-node1.example.com   kubernetes.io/hostname=ose3-node1.example.com   Ready
ose3-node2.example.com   kubernetes.io/hostname=ose3-node2.example.com   Ready

[vagrant@ose3-master log]$ oc get node ose3-node1.example.com
NAME                     LABELS                                          STATUS
ose3-node1.example.com   kubernetes.io/hostname=ose3-node1.example.com   Ready

[vagrant@ose3-master log]$ oc get node ose3-node2.example.com
NAME                     LABELS                                          STATUS
ose3-node2.example.com   kubernetes.io/hostname=ose3-node2.example.com   Ready
{% endhighlight %}

Also I get an _ok_ from the healthcheck

{% highlight bash %}
vagrant ssh master
curl -k https://ose3-master.example.com:8443/healthz
ok
{% endhighlight %}

### "Workaround" for the Hang ###

I'm not yet aware what side effects this may create, but it allows the playbook continue.

- Comment out the last 3 tasks in the [openshift_node role](https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_node/tasks/main.yml).

	- _Check scheduleable state_
	- _Handle unscheduleable node_
	- _Handle scheduleable node_

- Re-run the provisioning step and Ansible

{% highlight bash %}
vagrant reload --provision
{% endhighlight %}

### Bug! Vagrant Hostnames ###

OK great that seemed to work, and the master can be reached at https://127.0.0.1:8443/console but how can I reach https://ose3-master.example.com:8443/console ?

For now I just added the name to my localhost line in `/etc/hosts`, but is there a better way to do this automatically with Vagrant?

{% highlight text %}
127.0.0.1	localhost ose3-master.example.com
{% endhighlight %}

## Configure OpenShift ##

_I haven't finished going through this bit_

Off to see [getting started](https://github.com/openshift/origin#getting-started). 

- Create a docker regitsry. _this fails_

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
