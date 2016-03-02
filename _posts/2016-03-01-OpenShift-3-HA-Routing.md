---
title: OpenShift High Availability - Routing
layout: post
tags:
 - openshift
 - docker
---

High availability of containers in OpenShift is baked into the cake thanks to [replication controllers](https://docs.openshift.com/enterprise/3.1/architecture/core_concepts/deployments.html#replication-controllers) and [service load balancing](https://docs.openshift.com/enterprise/3.1/architecture/core_concepts/pods_and_services.html#services), but there are plenty of other single points of failure. Here is how to eliminate many of those.

_I'm still working on this post_

# OpenShift High Availability Configuration #

Some backgrouund docs:

- [OpenShift Router Concept](https://docs.openshift.com/enterprise/3.1/architecture/core_concepts/routes.html#routers)
- [OpenShift Router Deployment](https://docs.openshift.com/enterprise/3.1/install_config/install/deploy_router.html)
- [OpenShift Highly Available Router](https://docs.openshift.com/enterprise/3.1/admin_guide/high_availability.html)
- [Load Balance of non-HTTP](https://github.com/kubernetes/contrib/tree/master/service-loadbalancer) is not yet available beyond node ports and ipfailover

# Host Inventory and Installation #

Of course you'll be doing [advanced install](https://docs.openshift.com/enterprise/3.1/install_config/install/advanced_install.html) which leverages the [OpenShift Ansible playbook](https://github.com/openshift/openshift-ansible).

Here is an overview of the hosts.

**Infrastructure Nodes**

Name                       | IP         | Labels
---------------------------|------------|----------------------------
ose-ha-node-01.example.com | 192.0.2.1  | _region=infra_, _zone=metal_
ose-ha-node-02.example.com | 192.0.2.2  | _region=infra_, _zone=metal_

**Application Nodes**

Name                       | IP         | Labels
---------------------------|------------|----------------------------
ose-ha-node-03.example.com | 192.0.2.3  | _region=primary_, _zone=rhev_
ose-ha-node-04.example.com | 192.0.2.4  | _region=primary_, _zone=rhev_
ose-ha-node-05.example.com | 192.0.2.5  | _region=primary_, _zone=rhev_

**Master Nodes**

Name                         | IP         | Labels
-----------------------------|------------|----------------------------
ose-ha-master-01.example.com | 192.0.2.21 | _region=infra_, _zone=rhev_
ose-ha-master-02.example.com | 192.0.2.22 | _region=infra_, _zone=rhev_
ose-ha-master-03.example.com | 192.0.2.23 | _region=infra_, _zone=rhev_

**Etcd Servers**

Name                       | IP
---------------------------|-----------
ose-ha-etcd-01.example.com | 192.0.2.31
ose-ha-etcd-02.example.com | 192.0.2.32
ose-ha-etcd-02.example.com | 192.0.2.33

**Load Balancer Servers**

These hosts run haproxy and front end the masters thanks to `openshift_master_cluster_hostname=ose-master.os.example.com` in the hosts file.

Name                     | IP
-------------------------|-----------
ose-ha-lb-01.example.com | 192.0.2.41
ose-ha-lb-02.example.com | 192.0.2.42

**Hosts Inventory File**

And here is the inventory file.

```ini
[OSEv3:children]
masters
nodes
etcd
lb
[OSEv3:vars]
ansible_ssh_user=root
debug_level=2
deployment_type=openshift-enterprise
use_cluster_metrics=true
openshift_master_metrics_public_url=https://metrics.os.example.com/hawkular/metrics
openshift_master_identity_providers=[{'name': 'my_ldap_provider', 'challenge': 'true', 'login': 'true', 'kind': 'LDAPPasswordIdentityProvider', 'attributes': {'id': ['dn'], 'email': ['mail'], 'name': ['cn'], 'preferredUsername': ['uid']}, 'bindDN': '', 'bindPassword': '', 'ca': '', 'insecure': 'true', 'url': 'ldap://ldap.example.com:389/ou=people,o=example.com?uid'}]
use_fluentd=true
openshift_master_cluster_method=native
openshift_master_cluster_hostname=ose-master.os.example.com
openshift_master_cluster_public_hostname=ose-master.os.example.com
osm_default_subdomain=os.example.com
osm_default_node_selector='region=primary'
openshift_router_selector='region=infra'
openshift_registry_selector='region=infra'
[masters]
ose-ha-master-[01:03].example.com
[etcd]
ose-ha-etcd-[01:03].example.com
[lb]
ose-ha-lb-01.example.com
ose-ha-lb-02.example.com
[nodes]
ose-ha-master-[01:03].example.com openshift_node_labels="{'region': 'infra', 'zone': 'rhev'}" openshift_schedulable=False
ose-ha-node-[01:02].example.com   openshift_node_labels="{'region': 'infra', 'zone': 'metal'}"
ose-ha-node-[03:05].example.com   openshift_node_labels="{'region': 'primary', 'zone': 'rhev'}"
```

## Perform the Install ##

Run [my prep playbook](Playbook-to-Prepare-for-OpenShift-Enterprise-3.1-Install.md) and then run the byo playbook to perform the actual install.


# Configuration #

## Initial DNS Configuration ##

Access to applications like `app-namespace.os.example.com` starts with a wildcard DNS A record in your domain, `*.os.example.com` pointing to a router pod. The router pod should be assigned to an infrastructure node since the container will be using the host port to attach ha-proxy to.

The DNS records should point to the `router` pods which are using the infrastructure node host ports. That means the DNS record should point to the IP of the infrastructure node(s). But what if that node fails? Don't worry about that just yet.

Using nsupdate and a key which is allowed to manipulate our `os.example.com` zone, let's insert a `*` wildcard, and  pointi the name `ose-master` at the IP of the first load balancer node (for now).

```bash
nsupdate -v -k os.example.com.key
    update add *.os.example.com          300 A 192.0.2.1
    update add ose-master.os.example.com 300 A 192.0.2.41
    send
    quit
```

## HA Routing ##

That can be fixed with a `ipfailover` service and floating IPs. The result will look like this:

[![OpenShift HA Routing](/images/thumb/openshift-ha-cluster-routing.png)](/images/openshift-ha-cluster-routing.png)

[Configure IP Failover](https://docs.openshift.com/enterprise/3.1/admin_guide/high_availability.html#configuring-ip-failover)

Create a HA router set for the application pods in the `primary` region. The routers will run on the schedulable nodes in the `infra` region.

OpenShift's ipfailover internally uses [keepalived](http://www.keepalived.org/), so ensure that multicast is enabled on the labeled nodes, specifically the [VRRP](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol) multicast IP address 224.0.0.18.

- Label the nodes so they can be selected for the service

```bash
oc label nodes ose-ha-node-0{1,2} "ha-router=primary"

# confirm the change
oc get nodes --selector='ha-router=primary'
NAME                         LABELS                                                                                       STATUS    AGE
ose-ha-node-01.example.com   ha-router=primary,kubernetes.io/hostname=ose-ha-node-01.example.com,region=infra,zone=rhev   Ready     3d
ose-ha-node-02.example.com   ha-router=primary,kubernetes.io/hostname=ose-ha-node-02.example.com,region=infra,zone=rhev   Ready     3d
```

- Use _router_ service account (or optionally create _ipfailover_ account) to create the router. Check that it exists.

```bash
oc get scc privileged -o json | jq .users
[
  "system:serviceaccount:default:registry",
  "system:serviceaccount:default:router",
  "system:serviceaccount:openshift-infra:build-controller"
]
```

- Start N replicas of the where N is count of infra nodes. Label the router with `ha-router=primary` # **TODO** add 3rd infra node and VIP @csochin

```bash
oadm router ha-router-primary \
    --replicas=2 \
    --selector="ha-router=primary" \
    --selector="region=infra" \
    --labels="ha-router=primary" \
    --credentials=/etc/origin/master/openshift-router.kubeconfig \
    --service-account=router
password for stats user admin has been set to cixBqxbXyz
DeploymentConfig "ha-router-primary" created
Service "ha-router-primary" created
```

Edit the deployment config for the HA router and add the default cert within `spec.containers.env.name[DEFAULT_CERTIFICATE]`

```bash
oc edit dc ha-router-primary
```

From the initial install, there will be a pre-existing router (_router-1_) holding the host ports (80,443) which precludes starting the ha router instances. Scale that old router down to 0 pods:

```bash
oc scale --replicas=0 rc router-1
```

Pick 2 IP addresses which will float between the 2 infra nodes and create a IP failover service.

**IP Failover Nodes**

IP Failover Service        | IPs                      | Labels
---------------------------|--------------------------|----------------------------
ipf-ha-router-primary      | 192.0.2.101, 192.0.2.102 | _ha-router=primary_

Create a IP failover configuration named `ipf-ha-router-primary` having N replicas equal to number nodes labeled `ha-router=primary`

```bash
oadm ipfailover ipf-ha-router-primary \
    --replicas=2 \
    --watch-port=80 \
    --selector="ha-router=primary" \
    --virtual-ips="192.0.2.101-102" \
    --credentials=/etc/origin/master/openshift-router.kubeconfig \
    --service-account=router \
    --create
```

# OpenShift HA DNS Configuration #

Wildcard DNS records point users to the OpenShift routers providing ingress to application services.

Update the DNS wildcard records to reflect the floating IPs instead of the infra nodes' primary IPs.

```bash
nsupdate -v -k os.example.com.key
    update delete *.os.example.com 300 A 192.0.2.1
    update add    *.os.example.com 300 A 192.0.2.101
    update delete *.os.example.com 300 A 192.0.2.2
    update add    *.os.example.com 300 A 192.0.2.102
    send
    quit
```

## Router SSL ##

**TODO**

This would have installed the default cert off the bat, but has not been tested:

```bash
# test. not ran
oadm router test-ha-router-primary \
    --replicas=2 \
    --selector="ha-router=primary" \
    --selector="region=infra" \
    --labels="ha-router=primary" \
    --credentials=/etc/origin/master/openshift-router.kubeconfig \
    --service-account=router \
    --default-cert=wildcard.example.router.pem \
    --dry-run \
    -o json \
    --create=false
```

## HA Master ##

**TODO**

Load balance the API endpoint. If a `lb` group is defined in the Ansible playbook inventory then a haproxy node will be setup to load balance the master. That instance is a SPoF.

### Registry ##

**TODO**

