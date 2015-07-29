---
title: Testing OpenShift Enterprise V3
layout: post
tags:
 - openshift
 - RHEL
 - RHEV
---

So much for testing [OpenShift Origin with Vagrant on OS X](/blog/2015/06/28/Testing-Openshift-Origin-V3-with-Ansible-and-Vagrant-on-OS-X). Let's evaluate OpenShift Enterprise v3 on RHEL! First go get yourself an eval license. The OpenShift VMs will run RHEL7.1 and ride on top of RHEV 3.4.

# Links #

- [Getting Started](https://access.redhat.com/products/openshift-enterprise-red-hat/get-started)
- [Docs](https://access.redhat.com/documentation/en-US/OpenShift_Enterprise/)
- [Overview](http://docs.openshift.com/enterprise/latest/admin_guide/overview.html)
- [Download](https://install.openshift.com/)
- [Prerequisites](https://docs.openshift.com/enterprise/3.0/admin_guide/install/prerequisites.html)

# Prereqs #

[Prereqs](https://docs.openshift.com/enterprise/3.0/admin_guide/install/prerequisites.html) are:

[Three VMs](https://docs.openshift.com/enterprise/3.0/admin_guide/install/prerequisites.html#system-requirements) running Stock RHEL7.1 with valid subscriptions:

- ose3-master 2vCPU, 8G RAM 30G disk
- ose3-node1 1vCPU, 8G RAM 15G disk 15G [second disk](https://docs.openshift.com/enterprise/3.0/admin_guide/install/prerequisites.html#configuring-docker-storage) for docker images
- ose3-node2 1vCPU, 8G RAM 15G disk 15G second disk for docker images

# Setup RHEL Subs #

- On _all 3 hosts_ Register VM with RedHat Subscription Manager

{% highlight bash %}
subscription-manager register --username=rhel-username --password=<password>

subscription-manager list --available
{% endhighlight %}

- On _all 3 hosts_ Find the pool ID of our eval license:

{% highlight bash %}
subscription-manager list --available
...
Subscription Name:   30 Day Self-Supported OpenShift Enterprise, 2 Cores Evaluation
Provides:            Red Hat Beta
                     Red Hat JBoss A-MQ Clients
                     Red Hat OpenShift Enterprise
                     Red Hat OpenShift Enterprise JBoss EAP add-on
                     JBoss Enterprise Application Platform
                     Red Hat OpenShift Enterprise Application Node
                     Red Hat Software Collections (for RHEL Server)
                     Red Hat OpenShift Enterprise JBoss A-MQ add-on
                     JBoss Enterprise Web Server
                     Red Hat OpenShift Enterprise JBoss FUSE add-on
                     Oracle Java (for RHEL Server)
                     Red Hat OpenShift Enterprise Client Tools
                     Red Hat Enterprise Linux Server
                     Red Hat Software Collections Beta (for RHEL Server)
                     Red Hat OpenShift Enterprise Infrastructure
SKU:                 SER0419
Contract:            10000000
Pool ID:             8a000000000000000000000000000000
Provides Management: Yes
Available:           3
Suggested:           1
Service Level:       Self-Support
Service Type:        L1-L3
Subscription Type:   Standard
Ends:                08/22/2015
System Type:         Physical
{% endhighlight %}

- On _all 3 hosts_ Attach server to that pool ID

{% highlight bash %}
subscription-manager attach --pool=8a000000000000000000000000000000
{% endhighlight %}

- On _all 3 hosts_: Adjust subscriptions

{% highlight bash %}
subscription-manager repos --disable="*"
subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-optional-rpms" \
    --enable="rhel-7-server-ose-3.0-rpms"
{% endhighlight %}

- On _all 3 hosts_: Install Prereqs

{% highlight bash %}
yum -y remove \
  NetworkManager
yum -y install \
  python python-virtualenv openssh-clients gcc \
  wget git net-tools bind-utils iptables-services bridge-utils \
  docker
yum -y update
{% endhighlight %}

# Configure OpenShift #

## Configure Docker Storage on Nodes ##

See [Configure Docker Storage](https://docs.openshift.com/enterprise/3.0/admin_guide/install/prerequisites.html#configuring-docker-storage)

- On _ose3-node1_ and on _ose3-node2_ create a LVM thin pool on the 2nd disk. This should be done before the installer playbook runs, and that is why we have to install docker by hand.

{% highlight bash %}
cat <<EOF > /etc/sysconfig/docker-storage-setup
DEVS=/dev/vdb
VG=docker-vg
EOF

docker-storage-setup
{% endhighlight %}

Example:

{% highlight text  %}
[root@ose3-node1 ~]# docker-storage-setup
0
Checking that no-one is using this disk right now ...
OK

Disk /dev/vdb: 31207 cylinders, 16 heads, 63 sectors/track
sfdisk:  /dev/vdb: unrecognized partition table type

Old situation:
sfdisk: No partitions found

New situation:
Units: sectors of 512 bytes, counting from 0

   Device Boot    Start       End   #sectors  Id  System
/dev/vdb1          2048  31457279   31455232  8e  Linux LVM
/dev/vdb2             0         -          0   0  Empty
/dev/vdb3             0         -          0   0  Empty
/dev/vdb4             0         -          0   0  Empty
Warning: partition 1 does not start at a cylinder boundary
Warning: partition 1 does not end at a cylinder boundary
Warning: no primary partition is marked bootable (active)
This does not matter for LILO, but the DOS MBR will not boot this disk.
Successfully wrote the new partition table

Re-reading the partition table ...

If you created or changed a DOS partition, /dev/foo7, say, then use dd(1)
to zero the first 512 bytes:  dd if=/dev/zero of=/dev/foo7 bs=512 count=1
(See fdisk(8).)
  Physical volume "/dev/vdb1" successfully created
  Volume group "docker-vg" successfully created
  Rounding up size to full physical extent 16.00 MiB
  Logical volume "docker-poolmeta" created.
  Logical volume "docker-pool" created.
  WARNING: Converting logical volume docker-vg/docker-pool and docker-vg/docker-poolmeta to pool's data and metadata volumes.
  THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
  Converted docker-vg/docker-pool to thin pool.
  Logical volume "docker-pool" changed.

[root@ose3-node1 ~]# cat /etc/sysconfig/docker-storage
DOCKER_STORAGE_OPTIONS=-s devicemapper --storage-opt dm.fs=xfs --storage-opt dm.thinpooldev=/dev/mapper/docker--vg-docker--pool

[root@ose3-node1 ~]# pvs
  PV         VG              Fmt  Attr PSize  PFree
  /dev/vda2  rhel_ose3-node1 lvm2 a--  14.51g 12.00m
  /dev/vdb1  docker-vg       lvm2 a--  15.00g  5.98g

[root@ose3-node1 ~]# vgs
  VG              #PV #LV #SN Attr   VSize  VFree
  docker-vg         1   1   0 wz--n- 15.00g  5.98g
  rhel_ose3-node1   1   2   0 wz--n- 14.51g 40.00m

[root@ose3-node1 ~]# lvs
  LV          VG              Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  docker-pool docker-vg       twi-a-t---  8.99g             0.00   0.24
  root        rhel_ose3-node1 -wi-ao---- 12.97g
  swap        rhel_ose3-node1 -wi-ao----  1.50g

[root@ose3-node1 ~]# lvdisplay docker-vg/docker-pool
  --- Logical volume ---
  LV Name                docker-pool
  VG Name                docker-vg
  LV UUID                7JhHzN-lPQc-WrOy-e9cJ-e7aA-50Sy-syDVaS
  LV Write Access        read/write
  LV Creation host, time ose3-node1, 2015-07-27 16:11:09 -0700
  LV Pool metadata       docker-pool_tmeta
  LV Pool data           docker-pool_tdata
  LV Status              available
  # open                 0
  LV Size                8.99 GiB
  Allocated pool data    0.00%
  Allocated metadata     0.24%
  Current LE             2301
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:4
{% endhighlight %}

**TODO**
- Configure [persistent storage](https://docs.openshift.com/enterprise/3.0/admin_guide/persistent_storage_nfs.html)

## Allow Access to Insecure Registries ##

- Enable insecure docker registries

The docs said to do this, but [ansible does this during the install to follow](https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_node/tasks/main.yml#L66). **Documentation bug to file?**

{% highlight bash %}
[root@ose3-node1 ~]# grep OPTIONS /etc/sysconfig/docker
OPTIONS='--insecure-registry=172.30.0.0/16 --selinux-enabled'
{% endhighlight %}

**TODO**
- fix up the registry security.

# Install OpenShift on Master #

This will [use Ansible](https://github.com/openshift/openshift-ansible) to install `openshift-node` on the nodes.

- On _ose3-master_: Setup ssh so the ansible installer can run in the next step. 

{% highlight bash %}
ssh-keygen
cp -p .ssh/id_rsa.pub .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
ssh-copy-id ose3-node1
ssh-copy-id ose3-node2
{% endhighlight %}

- On _ose3-master_: Try to install OSE

{% highlight text %}
sh <(curl -s https://install.openshift.com/ose)

# blah blah blah
ose3-node1.example.com,1.2.3.94,1.2.3.94,ose3-node1.example.com,ose3-node1.example.com
ose3-master.example.com,1.2.3.124,1.2.3.124,ose3-master.example.com,ose3-master.example.com
ose3-node2.example.com,1.2.3.123,1.2.3.123,ose3-node2.example.com,ose3-node2.example.com

# Everything after this line is ignored.

Format:

installation host,IP,public IP,hostname,public hostname

Notes:
 * The installation host is the hostname from the installer's perspective.
 * The IP of the host should be the internal IP of the instance.
 * The public IP should be the externally accessible IP associated with the instance
 * The hostname should resolve to the internal IP from the instances
   themselves.
 * The public hostname should resolve to the external ip from hosts outside of
   the cloud.
The installation was successful!

If this is your first time installing please take a look at the Administrator
Guide for advanced options related to routing, storage, authentication and much
more:

http://docs.openshift.com/enterprise/latest/admin_guide/overview.html
{% endhighlight %}

# Configure OpenShift #

See [Admin Guide Overview](http://docs.openshift.com/enterprise/latest/admin_guide/overview.html)

## Create a Docker Registry ##

- Create a Docker Registry on _ose3-master_

{% highlight text %}
[root@ose3-master ~]#  oadm registry --config=/etc/openshift/master/admin.kubeconfig \
    --credentials=/etc/openshift/master/openshift-registry.kubeconfig \
    --images='registry.access.redhat.com/openshift3/ose-${component}:${version}'

[root@ose3-master ~]# oc get pods
NAME                      READY     REASON    RESTARTS   AGE
docker-registry-1-g348m   1/1       Running   0          2m

[root@ose3-master ~]# oc get rc
CONTROLLER          CONTAINER(S)   IMAGE(S)                                                             SELECTOR                                                                                REPLICAS
docker-registry-1   registry       registry.access.redhat.com/openshift3/ose-docker-registry:v3.0.0.1   deployment=docker-registry-1,deploymentconfig=docker-registry,docker-registry=default   1

[root@ose3-master ~]# oc logs docker-registry-1-g348m
time="2015-07-24T19:11:46-04:00" level=info msg="version=v2.0.0+unknown"
time="2015-07-24T19:11:46-04:00" level=info msg="redis not configured" instance.id=17d8ba37-16c4-4bc9-9dff-3ce4337c16d8
time="2015-07-24T19:11:46-04:00" level=info msg="using inmemory layerinfo cache" instance.id=17d8ba37-16c4-4bc9-9dff-3ce4337c16d8
time="2015-07-24T19:11:46-04:00" level=info msg="Using OpenShift Auth handler"
time="2015-07-24T19:11:46-04:00" level=info msg="listening on :5000" instance.id=17d8ba37-16c4-4bc9-9dff-3ce4337c16d8
time="2015-07-24T19:11:46-04:00" level=info msg="Starting upload purge in 18m0s" instance.id=17d8ba37-16c4-4bc9-9dff-3ce4337c16d8
time="2015-07-24T19:29:46-04:00" level=info msg="PurgeUploads starting: olderThan=2015-07-17 19:29:46.86046945 -0400 EDT, actuallyDelete=true"
time="2015-07-24T19:29:46-04:00" level=info msg=Base.List trace.duration=41.839µs trace.file="/builddir/build/BUILD/openshift-git-4.eab4c86/_thirdpartyhacks/src/github.com/docker/distribution/registry/storage/driver/base/base.go" trace.func="github.com/docker/distribution/registry/storage/driver/base.(*Base).List" trace.id=8c3d1799-e480-4dc3-9d2d-72c6f728d829 trace.line=123
time="2015-07-24T19:29:46-04:00" level=info msg="Purge uploads finished.  Num deleted=0, num errors=1"
time="2015-07-24T19:29:46-04:00" level=info msg="Starting upload purge in 24h0m0s" instance.id=17d8ba37-16c4-4bc9-9dff-3ce4337c16d8
{% endhighlight %}

## Deploy a router ##

See [Deploy Router](https://docs.openshift.com/enterprise/3.0/admin_guide/install/deploy_router.html)

- To see what the router would look like run:
{% highlight bash %}
oadm router -o yaml \
    --credentials='/etc/openshift/master/openshift-router.kubeconfig' \
    --images='registry.access.redhat.com/openshift3/ose-${component}:${version}'
{% endhighlight %}

- _On ose3-master_  create a router and note the stats password.

{% highlight bash %}
oadm router router1 --replicas=2 \
    --credentials='/etc/openshift/master/openshift-router.kubeconfig' \
    --images='registry.access.redhat.com/openshift3/ose-${component}:${version}'
password for stats user admin has been set to noB000002X
deploymentconfigs/router1
services/router1
{% endhighlight %}

**NOTE!**

Can not access [haproxy stats](http://ose3-node1.example.com:1936) unless you turn off iptables **BUG**

{% highlight text  %}
[root@ose3-master ~]# oc get pods
NAME                      READY     REASON    RESTARTS   AGE
docker-registry-1-udgkk   1/1       Running   0          45m
router1-1-miamx           1/1       Running   0          19m
router1-1-y1wk8           1/1       Running   0          20m
{% endhighlight %}

# First Steps #

- [First Steps](https://docs.openshift.com/enterprise/3.0/admin_guide/install/first_steps.html)

## Create Image Streams ##

_It looks like this were already done by the playbook_

Create the core set of image streams, which use RHEL 7 based images:

{% highlight bash %}
oc create -f \
    /usr/share/openshift/examples/image-streams/image-streams-rhel7.json \
    -n openshift
{% endhighlight %}

## Create Database Service Templates ##

_It looks like this were already done by the playbook_

{% highlight bash %}
oc create -f \
    /usr/share/openshift/examples/db-templates -n openshift
{% endhighlight %}

## Create Quick Start Templates ##

_It looks like this were already done by the playbook_

{% highlight bash %}
oc create -f \
    /usr/share/openshift/examples/quickstart-templates -n openshift
{% endhighlight %}

# Configure Authentication Bypass #

- [Auth](https://docs.openshift.com/enterprise/3.0/admin_guide/configuring_authentication.html)

By default `/etc/openshift/master/master-config.yaml` denys all. Let's allow any username to work regardless of password.

{% highlight diff %}
--- master-config.yaml  2015-07-27 17:54:16.403491361 -0700
+++ master-config.yaml.bak      2015-07-27 16:35:59.777477091 -0700
@@ -82,12 +82,12 @@
   grantConfig:
     method: auto
   identityProviders:
-  - name: my_allow_provider
-    challenge: true
-    login: true
+  - name: deny_all
+    challenge: True
+    login: True
     provider:
       apiVersion: v1
-      kind: AllowAllPasswordIdentityProvider
+      kind: DenyAllPasswordIdentityProvider
   masterPublicURL: https://ose3-master.example.com:8443
   masterURL: https://ose3-master.example.com:8443
   sessionConfig:
{% endhighlight %}

- Now restart the server on ose3-master.

{% highlight bash %}
systemctl restart openshift-master
{% endhighlight %}

## Create Project ##

- [Login to Console](https://ose3-master.example.com:8443)
- Create a project called _eval_
- Create a _cakephp example app_

# Test out Developer UI #

[What’s Next?](https://docs.openshift.com/enterprise/3.0/admin_guide/install/first_steps.html#what-s-next)

With these artifacts created, developers can now log into the web console and follow the flow for creating from a template. Any of the database or application templates can be selected to create a running database service or application in the current project. Note that some of the application templates define their own database services as well.

The example applications are all built out of GitHub repositories which are referenced in the templates by default, as seen in the SOURCE_REPOSITORY_URL parameter value. Those repositories can be forked, and the fork can be provided as the SOURCE_REPOSITORY_URL parameter value when creating from the templates. This allows developers to experiment with creating their own applications.

You can direct your developers to the Using the QuickStart Templates section in the Developer Guide for these instructions.

- https://ose3-master.example.com:8443

