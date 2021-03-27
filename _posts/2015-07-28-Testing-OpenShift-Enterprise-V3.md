---
title: Testing OpenShift Enterprise V3
layout: post
tags:
 - openshift
 - OCP3
 - RHEL
---

So much for testing [OpenShift Origin with Vagrant on OS X](/blog/2015/06/28/Testing-Openshift-Origin-V3-with-Ansible-and-Vagrant-on-OS-X), because [it does not work yet](https://github.com/openshift/openshift-ansible/issues/391). Let's evaluate OpenShift Enterprise v3 on RHEL! First go get yourself an eval license. The OpenShift VMs will run RHEL7.1 and ride on top of [RHEV](https://access.redhat.com/products/red-hat-enterprise-virtualization/).

# Documentation #

First off, here are some starting points to get oriented and acquainted with OpenShift.

**Docs**

- [Getting Started](https://access.redhat.com/products/openshift-enterprise-red-hat/get-started)
- [Docs](https://access.redhat.com/documentation/en-US/OpenShift_Enterprise/)
- [Overview](http://docs.openshift.com/enterprise/latest/admin_guide/overview.html)
- [Training](https://github.com/openshift/training)
- [Download](https://install.openshift.com/)
- [Prerequisites](https://docs.openshift.com/enterprise/3.1/admin_guide/install/prerequisites.html)
- [OpenShift Enterprise 3 Architecture Guide - planning, deployment and operation of an Open Source Platform as a Service](https://access.redhat.com/articles/1755133)
- [Load Balancing](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Load_Balancer_Administration/ch-lvs-overview-VSA.html)

**Videos**

- [OpenShift Channel on Youtube](https://www.youtube.com/channel/UCZKMj3YI0wP-kq4QYpaKdEA)
- [OpenShift Commons Briefing #15: OpenShift 3 Beta 4 Training on Operations Workflow](https://www.youtube.com/watch?t=67&v=nqf9ZBqVIQM) provides a great walk through of install and basic overview of the compnents

# Installation #

## Prereqs ##

There are [several prereqs](https://docs.openshift.com/enterprise/3.1/admin_guide/install/prerequisites.html) to meet before installation can begin. There are a few items to identify and used to create Ansible variables before installation.

Some Ansible variables we need to define:

- `openshift_master_portal_net`
- `osm_cluster_network_cidr`
- `osm_default_subdomain`

### VMs ###

Four [virtual machines](https://docs.openshift.com/enterprise/3.1/admin_guide/install/prerequisites.html#system-requirements) running Stock RHEL 7 with valid subscriptions:

These are higher than the min requirements, but it's what I'm going to use.

VM            | CPU | Mem | Disk(s)
--------------|-----|-----|----------
ose-master-01 | 2   | 16G | 30G (20G _/mnt/registry_?), 50G [second disk](https://docs.openshift.com/enterprise/3.1/admin_guide/install/prerequisites.html#configuring-docker-storage) for docker images
ose-node-01   | 2   | 16G | 30G OS, 50G Docker
ose-node-02   | 2   | 16G | 30G OS, 50G Docker
ose-node-03   | 2   | 16G | 30G OS, 50G Docker

## Storage ##

Docker on the node will use a [thin provisioned](http://unpoucode.blogspot.com.es/2015/06/docker-and-devicemappers-thinpool-in.html) LVM volume group for ephemeral container filesystems under `/var/lib/docker`.

An NFS export to be mounted by nodes for creation of [persistent volumes](https://docs.openshift.com/enterprise/3.1/architecture/additional_concepts/storage.html) and for persistent docker registry.

[Persistent storage using NFS](https://docs.openshift.com/enterprise/3.1/install_config/persistent_storage/persistent_storage_nfs.html) requires NFSv4 for SELinux. The [kube_nfs_volumes](https://github.com/openshift/openshift-ansible/tree/master/roles/kube_nfs_volumes) role can automate the creation of `persistent volumes`.

## Networks ##

Network Name               | Default         |  Ansible Variable             | Description
---------------------------|-----------------|-------------------------------|-----------------
`portal_net`               | _172.30.0.0/16_ | `openshift_master_portal_net` | Home for Services / Load Balancers. This should be routable in your organization.
`sdn_cluster_network_cidr` | _10.1.0.0/16_   | `osm_cluster_network_cidr`    | Docker Network. One `/24` allocated per node by default. SDN manages this network with Open vSwitch and VxLAN.

## DNS ##

Pick a wildcard subdomain domain and assumes.

 Default          |  Ansible Variable             | Description
------------------|-------------------------------|-----------------
 _cloudapps.com_  | `osm_default_subdomain`       | Subdomain to place application [routes](https://access.redhat.com/documentation/en/openshift-enterprise/version-3.1/openshift-enterprise-30-architecture/chapter-3-core-concepts#routes) in

Maybe pick multiple?
Perhaps we should use os.example.com as the "TLD" and use subdomain per env/cluster like this:

- *.dev.os.example.com.  300 IN  A <ip_of_openshift_router>
- *.test.os.example.com. 300 IN  A <ip_of_openshift_router>
- *.prod.os.example.com. 300 IN  A <ip_of_openshift_router>

# VM Installation #

## Setup RHEL Subs ##

- On _all the hosts_ Register VM with RedHat Subscription Manager

```bash
subscription-manager register --username=rhel-username --password=<password>

subscription-manager list --available
```

- On _all the hosts_ find the pool ID of the Red Hat OpenShift Enterprise license:

```bash
[root@ose-master-01 ~]# subscription-manager list --available
+-------------------------------------------+
    Available Subscriptions
+-------------------------------------------+
Subscription Name:   OpenShift Enterprise, Standard (1-2 Sockets)
Provides:            Red Hat Beta
                     Red Hat OpenShift Enterprise
                     Red Hat OpenShift Enterprise Application Node
                     Red Hat Software Collections (for RHEL Server)
                     JBoss Enterprise Web Server
                     Oracle Java (for RHEL Server)
                     Red Hat OpenShift Enterprise Client Tools
                     Red Hat Enterprise Linux Server
                     Red Hat Software Collections Beta (for RHEL Server)
                     Red Hat Enterprise Linux Atomic Host
SKU:                 MCT2863
Contract:            10000001
Pool ID:             8000000000000000000000000000000a
Provides Management: No
Available:           3
Suggested:           1
Service Level:       Standard
Service Type:        L1-L3
Subscription Type:   Stackable
Ends:                11/29/2016
System Type:         Physical

Subscription Name:   OpenShift Enterprise Broker Infrastructure
Provides:            Red Hat Beta
                     Red Hat OpenShift Enterprise
                     Red Hat Software Collections (for RHEL Server)
                     Oracle Java (for RHEL Server)
                     Red Hat OpenShift Enterprise Client Tools
                     Red Hat Enterprise Linux Server
                     Red Hat Software Collections Beta (for RHEL Server)
                     Red Hat Enterprise Linux Atomic Host
                     Red Hat OpenShift Enterprise Infrastructure
SKU:                 MCT2741
Contract:            10000001
Pool ID:             80000000000000000000000000000001
Provides Management: Yes
Available:           2
Suggested:           1
Service Level:       Layered
Service Type:        L1-L3
Subscription Type:   Standard
Ends:                11/29/2016
System Type:         Physical
...
```

- On all the hosts, attach to that pool ID

```bash
subscription-manager attach --pool=8a000000000000000000000000000000
```

- On _all 3 hosts_: Adjust subscriptions

```bash
subscription-manager repos --disable="*"
subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-optional-rpms" \
    --enable="rhel-7-server-ose-3.0-rpms"
```

- On _all 3 hosts_: Install Prereqs

```bash
yum -y remove \
  NetworkManager
# install reqs
yum -y install \
  python python-virtualenv openssh-clients gcc \
  wget git net-tools bind-utils iptables-services bridge-utils \
  docker
# install life enhancers
yum -y install \
  vim-enhanced deltarpm bash-completion tmux
yum -y update
```

It may be helpful to keep journald logs around across reboots while we are still testing.

```bash
mkdir /var/log/journal
systemctl restart systemd-journald
```

The following units may produce some interesting logs (`journalctl -f -u $unit`).

- openshift-master
- openshift-node
- openshift-sdn-master
- openshift-sdn-node

# Configure OpenShift #

## Configure Docker Storage on Nodes ##

See [Configure Docker Storage](https://docs.openshift.com/enterprise/3.0/admin_guide/install/prerequisites.html#configuring-docker-storage)

- On _ose3-node1_ and on _ose3-node2_ create a LVM thin pool on the 2nd disk. This should be done before the installer playbook runs, and that is why we have to install docker by hand.

```bash
cat <<EOF > /etc/sysconfig/docker-storage-setup
DEVS=/dev/vdb
VG=docker-vg
EOF

docker-storage-setup
```

Example:

```text
root@ose-node-01 ~]# cat <<EOF > /etc/sysconfig/docker-storage-setup
> DEVS=/dev/vdb
> VG=docker-vg
> EOF
[root@ose-node-01 ~]#
[root@ose-node-01 ~]# docker-storage-setup
Checking that no-one is using this disk right now ...
OK

Disk /dev/vdb: 104025 cylinders, 16 heads, 63 sectors/track
sfdisk:  /dev/vdb: unrecognized partition table type

Old situation:
sfdisk: No partitions found

New situation:
Units: sectors of 512 bytes, counting from 0

   Device Boot    Start       End   #sectors  Id  System
/dev/vdb1          2048 104857599  104855552  8e  Linux LVM
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
  Rounding up size to full physical extent 52.00 MiB
  Logical volume "docker-poolmeta" created.
  Logical volume "docker-pool" created.
  WARNING: Converting logical volume docker-vg/docker-pool and docker-vg/docker-poolmeta to pool's data and metadata volumes.
  THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
  Converted docker-vg/docker-pool to thin pool.
  Logical volume "docker-pool" changed.

[root@ose-node-01 ~]# cat /etc/sysconfig/docker-storage
DOCKER_STORAGE_OPTIONS=--storage-driver devicemapper --storage-opt dm.fs=xfs --storage-opt dm.thinpooldev=/dev/mapper/docker--vg-docker--pool

[root@ose-node-01 ~]# pvs
  PV         VG        Fmt  Attr PSize  PFree
  /dev/vda2  rhel      lvm2 a--  29.51g 40.00m
  /dev/vdb1  docker-vg lvm2 a--  50.00g 29.92g

[root@ose-node-01 ~]# vgs
  VG        #PV #LV #SN Attr   VSize  VFree
  docker-vg   1   1   0 wz--n- 50.00g 29.92g
  rhel        1   2   0 wz--n- 29.51g 40.00m

[root@ose-node-01 ~]# lvs
  LV          VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  docker-pool docker-vg twi-a-t--- 19.98g             0.00   0.08
  root        rhel      -wi-ao---- 26.47g
  swap        rhel      -wi-ao----  3.00g
```

The docker-pool volume should be 60% of the available volume group and will grow to fill the volume group via LVM monitoring.

```text
[root@ose-node-01 ~]# lvdisplay docker-vg/docker-pool
  --- Logical volume ---
  LV Name                docker-pool
  VG Name                docker-vg
  LV UUID                enMTwp-s9do-uKCi-3r9w-90bQ-Ht62-4civTW
  LV Write Access        read/write
  LV Creation host, time ose-node-01, 2015-11-15 21:23:09 -0800
  LV Pool metadata       docker-pool_tmeta
  LV Pool data           docker-pool_tdata
  LV Status              available
  # open                 0
  LV Size                19.98 GiB
  Allocated pool data    0.00%
  Allocated metadata     0.08%
  Current LE             5114
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:4
```

**TODO**
- Configure [persistent storage](https://docs.openshift.com/enterprise/3.0/admin_guide/persistent_storage_nfs.html)

## Allow Access to Insecure Registries ##

- Enable insecure docker registries

The docs said to do this, but [ansible does this during the install to follow](https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_node/tasks/main.yml#L66). **Documentation bug to file?**

```bash
[root@ose3-node1 ~]# grep OPTIONS /etc/sysconfig/docker
OPTIONS='--insecure-registry=172.30.0.0/16 --selinux-enabled'
```

Skip the above for the moment.

# Perform Advanced Ansible-based Install #

Now proceed with the OpenShift [advanced installation method](https://docs.openshift.com/enterprise/3.1/install_config/install/advanced_install.html).

- Clone the installer

```
git clone https://github.com/openshift/openshift-ansible
```

- Install Ansible from EPEL

```
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install ansible
```

Setup ssh on the master, such that the ansible installer can run in the next step. 

```bash
ssh-keygen
cp -p .ssh/id_rsa.pub .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
for h in `seq -f 'ose-node-%02g' 1 3`; do
  ssh-copy-id $h
done
```

## Create Ansible Inventory File ##

Based on the [byo/inventory/hosts.example](https://github.com/openshift/openshift-ansible/blob/master/inventory/byo/hosts.example)
example inventory and advice [here](https://docs.openshift.com/enterprise/3.1/install_config/install/advanced_install.html#configuring-ansible), 
we'll make our inventory own and place it in `/etc/ansible/hosts` on the master.

We'll have to identify values for use by the playbook such as our Portral Network  default subdomain etc.


## Install Openshift ##

```bash
# run this twice to validate connectivity cache the ssh keys answer yes repeatedly
[root@ose-master-01 ~]# ansible all -m ping
[root@ose-master-01 ~]# ansible all -m ping
ose-node-02.example.com | success >> {
    "changed": false,
    "ping": "pong"
}

ose-master-01.example.com | success >> {
    "changed": false,
    "ping": "pong"
}

ose-node-01.example.com | success >> {
    "changed": false,
    "ping": "pong"
}

ose-node-03.example.com | success >> {
    "changed": false,
    "ping": "pong"
}
```

- Run the _bring your own_ playbook

```bash
ansible-playbook openshift-ansible/playbooks/byo/config.yml
```

Look around a big

```text
[root@ose-master-01 ~]# oc get nodes
NAME                      LABELS                                                              STATUS                     AGE
ose-master-01.example.com   kubernetes.io/hostname=1.1.1.124,region=infra,zone=default     Ready,SchedulingDisabled   5m
ose-node-01.example.com     kubernetes.io/hostname=1.1.1.94,region=primary,zone=default    Ready                      5m
ose-node-02.example.com     kubernetes.io/hostname=1.1.1.123,region=primary,zone=default   Ready                      5m
ose-node-03.example.com     kubernetes.io/hostname=1.1.1.187,region=primary,zone=default   Ready                      5m

[root@ose-master-01 ~]# oc get endpoints
NAME         ENDPOINTS             AGE
kubernetes   1.1.1.124:8443   9m

[root@ose-master-01 ~]# oc get namespaces
NAME              LABELS    STATUS    AGE
default           <none>    Active    9m
openshift         <none>    Active    9m
openshift-infra   <none>    Active    9m

[root@ose-master-01 ~]# oc get services
NAME         CLUSTER_IP    EXTERNAL_IP   PORT(S)   SELECTOR   AGE
kubernetes   172.30.1.1   <none>        443/TCP   <none>     9m
```


# Configure OpenShift #

See [Admin Guide Overview](http://docs.openshift.com/enterprise/latest/admin_guide/overview.html)

## Create a Docker Registry ##

# TODO LEFT OFF HERE


- [Create a Docker Registry](https://docs.openshift.com/enterprise/3.1/install_config/install/docker_registry.html) on the master

```text
[root@ose-master-01 ~]# oadm registry \
>     --service-account=registry \
>     --config=/etc/openshift/master/admin.kubeconfig \
>     --credentials=/etc/openshift/master/openshift-registry.kubeconfig \
>     --images='registry.access.redhat.com/openshift3/ose-${component}:${version}' \
>     --selector='region=infra'
deploymentconfigs/docker-registry
services/docker-registry
# this following line would make the master /mnt/registry available in the registry container at /registery
#    --mount-host=/mnt/registry

# Use enterprise NFS storage backend
$ oc volume deploymentconfigs/docker-registry \
     --add --overwrite --name=registry-storage --mount-path=/registry \
     --source='{"nfs": { "server": "big_nfs_server_fqdn", "path": "/path/to/export"}}'

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
```

- TODO (maybe)

Create a new service account in the default project for the registry to run as. The following example creates a service account named registry:

```
```text
[root@master ~]# echo \
'{"kind":"ServiceAccount","apiVersion":"v1","metadata":{"name":"registry"}}' \
  | oc create ­n default ­f ­
[root@master ~]# echo \
'{"kind":"ServiceAccount","apiVersion":"v1","metadata":{"name":"router"}}' \
  | oc create ­f ­
[root@master ~]# oc edit scc privileged
...
users:
­ system:serviceaccount:openshift­infra:build­controller
­ system:serviceaccount:default:registry
­ system:serviceaccount:default:router
```


## Deploy a router ##

Now [Deploy](https://docs.openshift.com/enterprise/3.1/admin_guide/install/deploy_router.html) a [Router](https://access.redhat.com/beta/documentation/en/openshift-enterprise-30-architecture/chapter-3-core-concepts#routes) which is basically a haproxy instance, but it possible to use an external load balancer like an F5 .

- To see what the router would look like run:

```bash
oadm router -o yaml \
    --credentials='/etc/openshift/master/openshift-router.kubeconfig' \
    --images='registry.access.redhat.com/openshift3/ose-${component}:${version}'
```

- _On ose3-master_  create a router and note the stats password.

```bash
oadm router router1 --replicas=2 \
    --credentials='/etc/openshift/master/openshift-router.kubeconfig' \
    --images='registry.access.redhat.com/openshift3/ose-${component}:${version}'
password for stats user admin has been set to noB000002X
deploymentconfigs/router1
services/router1
```

```text
[root@ose3-master ~]# oc get pods
NAME                      READY     REASON    RESTARTS   AGE
docker-registry-1-udgkk   1/1       Running   0          45m
router1-1-miamx           1/1       Running   0          19m
router1-1-y1wk8           1/1       Running   0          20m
```

# First Steps #

## Create Examples ##

[First Steps](https://docs.openshift.com/enterprise/3.0/admin_guide/install/first_steps.html) doc describes how to import the following, but the playbook will already do this for you.

- Create the core set of image streams, which use RHEL 7 based images:

```bash
oc create -f \
    /usr/share/openshift/examples/image-streams/image-streams-rhel7.json \
    -n openshift
```

- Create Database Service Templates

```bash
oc create -f \
    /usr/share/openshift/examples/db-templates -n openshift
```

- Create Quick Start Templates

```bash
oc create -f \
    /usr/share/openshift/examples/quickstart-templates -n openshift
```

## Configure Authentication Bypass ##

- [Auth](https://docs.openshift.com/enterprise/3.1/admin_guide/configuring_authentication.html)

By default `/etc/openshift/master/master-config.yaml` denys all. Let's allow any username to work regardless of password.
Set this value in the ansible hosts file before installation to allow logins for any user regardless of password. Obviously, this is only appropriate for a testing environment.

```text
# Allow all auth
openshift_master_identity_providers=[{'name': 'allow_all', 'login': 'true', 'challenge': 'true', 'kind': 'AllowAllPasswordIdentityProvider'}]
```

- Now restart the server on ose3-master.

```bash
systemctl restart openshift-master
```

# Grok Some OpenShift Concepts #

*Users*

The `oc whoami` command will tell you what user you are. The root user is the `cluster-admin` with super ports.

*Projects*

Projects are essentially name spaces and define resources and quoatas for teams.

:changes



# Test the Developer UI #

## Create Project ##

- [Login to Console](https://ose3-master.example.com:8443)
- Create a project called _eval_
- Create a _cakephp example app_

[What’s Next?](https://docs.openshift.com/enterprise/3.1/admin_guide/install/first_steps.html#what-s-next)

With these artifacts created, developers can now log into the web console and follow the flow for creating from a template. Any of the database or application templates can be selected to create a running database service or application in the current project. Note that some of the application templates define their own database services as well.

The example applications are all built out of GitHub repositories which are referenced in the templates by default, as seen in the SOURCE_REPOSITORY_URL parameter value. Those repositories can be forked, and the fork can be provided as the SOURCE_REPOSITORY_URL parameter value when creating from the templates. This allows developers to experiment with creating their own applications.

You can direct your developers to the Using the QuickStart Templates section in the Developer Guide for these instructions.

- https://ose3-master.example.com:8443

## CLient Access ##

Download the client [from redhat](https://access.redhat.com/downloads/content/290/ver=3.0.0.0/rhel---7/3.0.2.0/x86_64/product-downloads) the client
