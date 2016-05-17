---
title: Upgrading OpenShift Enterprise from 3.1 to 3.2
layout: post
tags:
 - kubernetes
 - openshift
 - OSE3.1
 - OSE3.2
---

Upgrading from OSE 3.1 to 3.2 using the [playbook](https://github.com/openshift/openshift-ansible/blob/master/playbooks/common/openshift-cluster/upgrades/v3_1_to_v3_2/upgrade.yml) went quite well for me, but there were a few issues to sort out.

The issues were related to:

- [ip failover](#ip-failover)
- [downtime during the upgrade](#downtime-during-upgrade)
- [updates to image streams](#image-stream-updates)
- [docker error messages](#docker-errors)
- [updated policy and role bindings](#update-policies-and-roles)
- [hawkular metrics](#hawkular-metrics)

# Upgrade Process #

Following the directions is pretty straight forward.

- Start by prepping the yum repos.

```yaml
---
# file: upgrade-prep.yml
# 
# After running this see:
# - https://docs.openshift.com/enterprise/latest/install_config/upgrading/automated_upgrades.html
# - https://docs.openshift.com/enterprise/latest/install_config/upgrading/manual_upgrades.html

- hosts: all
  gather_facts: yes

  vars:
    ose_ver_old: 3.1
    ose_ver_new: 3.2

  tasks:

  - name: enable yum repos for upgrade version
    command: '/usr/sbin/subscription-manager repos --disable="rhel-7-server-ose-{{ose_ver_old}}-rpms" --enable="rhel-7-server-ose-{{ose_ver_new}}-rpms"'

  - name: update openshift-utils
    yum:
      name: atomic-openshift-utils
      state: latest
```

- Then run the upgrade

```bash
ansible-playbook -i hosts openshift-ansible/playbooks/byo/openshift-cluster/upgrades/v3_1_to_v3_2/upgrade.yml | \
  tee ansible-upgrade-$(date +%Y%m%d-%H%M).log
```

- 

# IP Failover #

I previously described the [native HA routing I am using](/blog/2016/03/01/OpenShift-3-HA-Routing).  

**Basically:**

```bash
$ oc get nodes -l "region=infra"
NAME                           STATUS                     AGE
ose-prod-master-01.example.com   Ready,SchedulingDisabled   71d
ose-prod-master-02.example.com   Ready,SchedulingDisabled   71d
ose-prod-master-03.example.com   Ready,SchedulingDisabled   71d
ose-prod-node-01.example.com     Ready                      71d
ose-prod-node-02.example.com     Ready                      71d
ose-prod-node-03.example.com     Ready                      71d

$ oc scale --replicas=0 dc router

$ oadm router ha-router-primary \
    --replicas=3 \
    --selector="ha-router=primary" \
    --selector="region=infra" \
    --labels="ha-router=primary" \
    --credentials=/etc/origin/master/openshift-router.kubeconfig \
    --default-cert=201602_router_wildcard.os.example.com.pem \
    --service-account=router

$ oadm ipfailover ipf-ha-router-primary \
    --replicas=3 \
    --watch-port=80 \
    --selector="ha-router=primary" \
    --virtual-ips="192.0.2.101-103" \
    --credentials=/etc/origin/master/openshift-router.kubeconfig \
    --service-account=router \
    --create
```

## IPF Image ##

The upgrade playbook updated the image used by the default router dc, but since I use Native HA with IPfailover I do not use that router. Mine is called `ha-router-primary`. I was able to manually update the image for `ha-router-primary` as described [here](https://docs.openshift.com/enterprise/3.2/install_config/upgrading/manual_upgrades.html#upgrading-the-router). However, that leaves the ipfailover pods. Presumably it should use `v3.2.0.20-3`, but I'm not sure how to validate that.

- **What image tag should I use for the `openshift3/ose-keepalived-ipfailover` image?**
- **How can I enumerate the tags to find this on my own?**

```bash
$ oc get dc -n default
NAME                    TRIGGERS       LATEST
docker-registry         ConfigChange   3
ha-router-primary       ConfigChange   2
ipf-ha-router-primary   ConfigChange   1
router                  ConfigChange   3      <-- unused. 0 replicas.

$ oc get dc ha-router-primary -n default -o json | jq '. | {name: .spec.template.spec.containers[].name, image:.spec.template.spec.containers[].image}'
{
  "image": "openshift3/ose-haproxy-router:v3.2.0.20-3",
  "name": "router"
}
$ oc get dc ipf-ha-router-primary -n default -o json | jq '. | {name: .spec.template.spec.containers[].name, image:.spec.template.spec.containers[].image}'
{
  "image": "openshift3/ose-keepalived-ipfailover:v3.1.1.6",
  "name": "ipf-ha-router-primary-keepalived"
}
```

On a related note, I'm hoping to see [this bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=1324594) integrated into the keepalived image soon, so [Sysdig](https://sysdig.com/) can be used on the infra nodes.

## IPfailover Readiness Check ##

The `oc status -v` command tells me I have no readiness check for my `ipf-ha-router-primary` dc. The [docs](https://docs.openshift.com/enterprise/3.2/admin_guide/high_availability.html) however do not mention a suitable check for ipfailover. 

- **Is there a suitable readiness check for ipfailover service?**

```
Warnings:
  * dc/docker-registry has no readiness probe to verify pods are ready to accept traffic or ensure deployment is successful.
    try: oc set probe dc/docker-registry --readiness ...
  * dc/ipf-ha-router-primary has no readiness probe to verify pods are ready to accept traffic or ensure deployment is successful.
    try: oc set probe dc/ipf-ha-router-primary --readiness ...
```

```
$ oc get dc/ipf-ha-router-primary
NAME                    REVISION   REPLICAS   TRIGGERED BY
ipf-ha-router-primary   1          3          config

$ oc describe dc/ipf-ha-router-primary
Name:           ipf-ha-router-primary
Created:        10 weeks ago
Labels:         ha-router=primary
Annotations:    <none>
Latest Version: 1
Selector:       ha-router=primary
Replicas:       3
Triggers:       Config
Strategy:       Recreate
Template:
  Labels:               ha-router=primary
  Service Account:      router
  Containers:
  ipf-ha-router-primary-keepalived:
    Image:      openshift3/ose-keepalived-ipfailover:v3.1.1.6
    Port:       1985/TCP
    QoS Tier:
      memory:   BestEffort
      cpu:      BestEffort
    Environment Variables:
      OPENSHIFT_CA_DATA:        -----BEGIN CERTIFICATE-----
...
WgXt+xTlODyXtB9CmeGWK+dqH3M8AgfRUJQ=
-----END CERTIFICATE-----

      OPENSHIFT_CERT_DATA:      -----BEGIN CERTIFICATE-----
...
MWdnBMd2iCXSsVJIafxc60o=
-----END CERTIFICATE-----

      OPENSHIFT_HA_CONFIG_NAME:         ipf-ha-router-primary
      OPENSHIFT_HA_MONITOR_PORT:        80
      OPENSHIFT_HA_NETWORK_INTERFACE:
      OPENSHIFT_HA_REPLICA_COUNT:       3
      OPENSHIFT_HA_USE_UNICAST:         false
      OPENSHIFT_HA_VIRTUAL_IPS:         192.0.2.101-103
      OPENSHIFT_INSECURE:               false
      OPENSHIFT_KEY_DATA:               -----BEGIN RSA PRIVATE KEY-----
...
xO/p48yGpN1JeqBQy7mlFQAIn1/4vBvfnjc/rljeqn19u01NuGkqJQ==
-----END RSA PRIVATE KEY-----

      OPENSHIFT_MASTER: https://master.os.example.com:8443
  Volumes:
  lib-modules:
    Type:       HostPath (bare host directory volume)
    Path:       /lib/modules

Deployment #1 (latest):
        Name:           ipf-ha-router-primary-1
        Created:        10 weeks ago
        Status:         Complete
        Replicas:       3 current / 3 desired
        Selector:       deployment=ipf-ha-router-primary-1,deploymentconfig=ipf-ha-router-primary,ha-router=primary
        Labels:         ha-router=primary,openshift.io/deployment-config.name=ipf-ha-router-primary
        Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed

No events.
```

## Downtime During Upgrade ##

The upgrade playbook ran fine from ose-prod-master-01 and took 11:49. I used the latest upstream playbook from Github as of a077b688e56374de31587bebcf4de20b22a6d51c. Output is attached.

```
localhost                  : ok=44   changed=0    unreachable=0    failed=0
ose-prod-etcd-01.example.com : ok=47   changed=4    unreachable=1    failed=0
ose-prod-etcd-02.example.com : ok=47   changed=4    unreachable=1    failed=0
ose-prod-etcd-03.example.com : ok=47   changed=4    unreachable=1    failed=0
ose-prod-lb-01.example.com   : ok=26   changed=2    unreachable=0    failed=0
ose-prod-master-01.example.com : ok=263  changed=39   unreachable=0    failed=0
ose-prod-master-02.example.com : ok=217  changed=34   unreachable=0    failed=0
ose-prod-master-03.example.com : ok=215  changed=31   unreachable=0    failed=0
ose-prod-node-01.example.com : ok=126  changed=20   unreachable=0    failed=0
ose-prod-node-02.example.com : ok=126  changed=20   unreachable=0    failed=0
ose-prod-node-03.example.com : ok=126  changed=20   unreachable=0    failed=0
ose-prod-node-04.example.com : ok=126  changed=20   unreachable=0    failed=0
ose-prod-node-05.example.com : ok=126  changed=20   unreachable=0    failed=0
ose-prod-node-06.example.com : ok=124  changed=17   unreachable=0    failed=0

real    11m49.852s
user    1m30.466s
sys     0m49.686s
```

During upgrade playbook run, I had pings going to my ipfailover VIPs each second. (`fping -l -p1000 < routers`). I saw 300 seconds of ping fails during upgrade.

- **Is there no way to avoid downtime of the router VIPs during upgrade?**

```
192.0.2.101 : xmt/rcv/%loss = 1497/1174/21%, min/avg/max = 0.14/0.31/84.1
192.0.2.102 : xmt/rcv/%loss = 1497/1181/21%, min/avg/max = 0.11/0.28/43.6
192.0.2.103 : xmt/rcv/%loss = 1497/1198/19%, min/avg/max = 0.12/0.27/22.0
```

# Image Stream Updates #

I still see 3.1.1 referenced by several images, and I'm still working out how to deduce the right image to use and to update them.

- **How do I update all OSE 3.1.1 vintage images?**

```
[root@ose-prod-node-05 ~]# docker images | grep 3.1.1
registry.access.redhat.com/openshift3/ose-docker-builder         v3.1.1.6            aefb1274aacc        6 weeks ago         442 MB
registry.access.redhat.com/openshift3/metrics-heapster           3.1.1               2ff00fc36375        10 weeks ago        230 MB
registry.access.redhat.com/openshift3/metrics-hawkular-metrics   3.1.1               5c02894a36cd        10 weeks ago        1.435 GB
registry.access.redhat.com/openshift3/metrics-cassandra          3.1.1               12d617a9aa1c        10 weeks ago        523 MB
registry.access.redhat.com/openshift3/ose-sti-builder            v3.1.1.6            75fa4984985a        11 weeks ago        441.9 MB
registry.access.redhat.com/openshift3/ose-deployer               v3.1.1.6            3bcee4233589        11 weeks ago        441.9 MB
registry.access.redhat.com/openshift3/ose-pod                    v3.1.1.6            ecfbff48161c        11 weeks ago        427.9 MB
```

# Docker Errors #

I am seeing the following error repeatedly:

```
May 17 11:33:23 ose-prod-node-05.example.com forward-journal[33427]: time="2016-05-17T11:33:23.255421438-07:00" level=error msg="Failed to get pwuid struct: user: unknown userid 4294967295"
May 17 11:33:23 ose-prod-node-05.example.com forward-journal[33427]: time="2016-05-17T11:33:23.263135054-07:00" level=error msg="Failed to get pwuid struct: user: unknown userid 4294967295"
May 17 11:33:23 ose-prod-node-05.example.com forward-journal[33427]: time="2016-05-17T11:33:23.283089775-07:00" level=error msg="Failed to get pwuid struct: user: unknown userid 4294967295"
May 17 11:33:23 ose-prod-node-05.example.com forward-journal[33427]: time="2016-05-17T11:33:23.696824533-07:00" level=error msg="Failed to get pwuid struct: user: unknown userid 4294967295"
May 17 11:33:23 ose-prod-node-05.example.com forward-journal[33427]: time="2016-05-17T11:33:23.959131064-07:00" level=error msg="Failed to get pwuid struct: user: unknown userid 4294967295"
May 17 11:33:23 ose-prod-node-05.example.com forward-journal[33427]: time="2016-05-17T11:33:23.994201790-07:00" level=error msg="Failed to get pwuid struct: user: unknown userid 4294967295"
```

This is even though [BZ#1275399](https://bugzilla.redhat.com/show_bug.cgi?id=1275399) indicates this was fixed by [Errata RHBA-2016-0536](https://rhn.redhat.com/errata/RHBA-2016-0536.html)

Wait, I still have the issue after an upgrade to OSE 3.2.0 and docker-1.9.1-40.el7.x86_64. [This PR](https://github.com/projectatomic/docker/pull/156) mentioned by [BZ#1335635](https://bugzilla.redhat.com/show_bug.cgi?id=1335635) indicates the fix should have been in Docker 1.9.1-40 and is coming in 1.10.


# Update Policies and Roles #

Run `oadm diagnostics` to do some sanity checks. Some results should be taken with a grain of salt.

I as prompted to use `oadm policy reconcile-cluster-role-bindings` and `oadm policy reconcile-cluster-roles`, but I have not done so with `--confirm` yet, because I've monkied with mine a bit. [See also](https://docs.openshift.com/enterprise/3.2/install_config/upgrading/manual_upgrades.html#updating-policy-definitions)

# Hawkular Metrics #

It used to be that you had to know the metrics URL and visit https://metrics.os.example.com and accept the cert so you could view the resource usage of your pods. There is now a nice link to this URL from the pod metrics page. However, even though I can vist the URL sucessfully, the metrics tab lists a _Forbidden_ error at the bottom, and no graphs. I suspect this may be related to the metrics running `openshift3/metrics-hawkular-metrics:3.1.1` still. This ties back to the [image stream updates](#image-stream-updates) issue.
