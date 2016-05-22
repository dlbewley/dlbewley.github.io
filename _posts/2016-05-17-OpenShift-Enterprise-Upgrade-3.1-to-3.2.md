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

- [ip failover](#ip-failover) had to be updated manually
- there was about 5 minutes [downtime during the upgrade](#downtime-during-upgrade)
- [updates to image streams](#image-stream-updates)
- [docker error messages](#docker-errors)
- [updated policy and role bindings](#update-cluster-policies-and-roles)
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

The upgrade playbook updated the image used by the default router dc, but since I use Native HA with IPfailover I do not use that router. Mine is called `ha-router-primary`. I was able to manually update the image for `ha-router-primary` as described [here](https://docs.openshift.com/enterprise/3.2/install_config/upgrading/manual_upgrades.html#upgrading-the-router).

```bash
$ oc get dc -n default
NAME                    TRIGGERS       LATEST
docker-registry         ConfigChange   3
ha-router-primary       ConfigChange   2
ipf-ha-router-primary   ConfigChange   1
router                  ConfigChange   3      <-- unused. 0 replicas.

$ oc get dc ha-router-primary -n default -o json | jq '. | {name: .spec.template.spec.containers[].name, image: .spec.template.spec.containers[].image}'
{
  "image": "openshift3/ose-haproxy-router:v3.2.0.20-3",
  "name": "router"
}
$ oc get dc ipf-ha-router-primary -n default -o json | jq '. | {name: .spec.template.spec.containers[].name, image: .spec.template.spec.containers[].image}'
{
  "image": "openshift3/ose-keepalived-ipfailover:v3.1.1.6",
  "name": "ipf-ha-router-primary-keepalived"
}
```

However, that leaves the ipfailover pods still at `v3.1.1.6`. Presumably it should use `v3.2.0.20-3`, but I'm not sure how to validate that.

- **What image tag should I use for the `openshift3/ose-keepalived-ipfailover` image?**

- **How can I enumerate the tags to find this on my own?**

Well, I can go search [registry.access.redhat.com](https://access.redhat.com/search/#/container-images), but the results don't list the version tags. Trying a `docker pull openshift3/ose-keepalived-ipfailover:v3.2.0.20-3` works, so let's do it. The [oc patch](https://github.com/openshift/origin/blob/master/docs/cli.md#oc-patch) command makes this "easy" if you like counting braces. 

```bash
$ oc patch dc ipf-ha-router-primary -p \
 '{"spec": {"template": {"spec": {"containers": [{"name": "ipf-ha-router-primary-keepalived", "image": "openshift3/ose-keepalived-ipfailover:v3.2.0.20-3"}]}}}}'
```

When the change is made it will be detected and the ipfailover pods will be recreated automatically. I didn't detect any downtime when doing this.

```bash
$ oc get dc ipf-ha-router-primary -n default -o json | jq '. | {name: .spec.template.spec.containers[].name, image: .spec.template.spec.containers[].image}'
{
  "name": "ipf-ha-router-primary-keepalived",
  "image": "openshift3/ose-keepalived-ipfailover:v3.2.0.20-3"
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

```bash
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

The upgrade playbook ran fine from ose-prod-master-01 and took 11:49. I used the latest upstream playbook cloned [from Github](https://github.com/openshift/openshift-ansible) rather than from `/usr/share/ansible/openshift-ansible`. Output is attached.

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

- **When with _get pwuid struct: user: unknown userid_ errors be fixed?**

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


# Update Cluster Policies and Roles #

Run `oadm diagnostics` to do some sanity checks. Some results should be taken with a grain of salt.

Cluster policies and roles changed, so update them to the latest. To see what needs to change run `oadm policy reconcile-cluster-roles` and `oadm policy reconcile-cluster-role-bindings`.
See [the docs](https://docs.openshift.com/enterprise/3.2/install_config/upgrading/manual_upgrades.html#updating-policy-definitions).

The [docs say](https://github.com/openshift/openshift-docs/issues/2131) to restart `atomic-openshift-master`, but I think that should `atomic-openshift-master-api


```bash
# examine output before continuing
$ oadm policy reconcile-cluster-roles
# make the changes suggested by above on one master
$ oadm policy reconcile-cluster-roles  --confirm
# restart api on all masters
$ systemctl restart atomic-openshift-master-api
# update policy role bindings
$ oadm policy reconcile-cluster-role-bindings \
    --exclude-groups=system:authenticated \
    --exclude-groups=system:authenticated:oauth \
    --exclude-groups=system:unauthenticated \
    --exclude-users=system:anonymous \
    --additive-only=true \
    --confirm
```

Even after this I'm seeing these warnings from `oadm diagnostics`

```
[Note] Running diagnostic: ClusterRoleBindings
       Description: Check that the default ClusterRoleBindings are present and contain the expected subjects

Info:  clusterrolebinding/cluster-admins has more subjects than expected.

       Use the `oadm policy reconcile-cluster-role-bindings` command to update the role binding to remove extra subjects.

Info:  clusterrolebinding/cluster-admins has extra subject {User  dlbewley    }.

Info:  clusterrolebinding/cluster-readers has more subjects than expected.

       Use the `oadm policy reconcile-cluster-role-bindings` command to update the role binding to remove extra subjects.

Info:  clusterrolebinding/cluster-readers has extra subject {ServiceAccount management-infra management-admin    }.
Info:  clusterrolebinding/cluster-readers has extra subject {ServiceAccount openshift-infra heapster    }.
Info:  clusterrolebinding/cluster-readers has extra subject {User  sysdig    }.
Info:  clusterrolebinding/cluster-readers has extra subject {ServiceAccount openshift-infra sysdig    }.

WARN:  [CRBD1003 from diagnostic ClusterRoleBindings@openshift/origin/pkg/diagnostics/cluster/rolebindings.go:83]
       clusterrolebinding/self-provisioners is missing expected subjects.

       Use the `oadm policy reconcile-cluster-role-bindings` command to update the role binding to include expected subjects.

Info:  clusterrolebinding/self-provisioners is missing subject {SystemGroup  system:authenticated:oauth    }.
Info:  clusterrolebinding/self-provisioners has extra subject {SystemGroup  system:authenticated    }.

WARN:  [CRBD1003 from diagnostic ClusterRoleBindings@openshift/origin/pkg/diagnostics/cluster/rolebindings.go:83]
       clusterrolebinding/system:build-strategy-docker-binding is missing expected subjects.

       Use the `oadm policy reconcile-cluster-role-bindings` command to update the role binding to include expected subjects.

Info:  clusterrolebinding/system:build-strategy-docker-binding is missing subject {SystemGroup  system:authenticated    }.

WARN:  [CRBD1003 from diagnostic ClusterRoleBindings@openshift/origin/pkg/diagnostics/cluster/rolebindings.go:83]
       clusterrolebinding/system:build-strategy-custom-binding is missing expected subjects.

       Use the `oadm policy reconcile-cluster-role-bindings` command to update the role binding to include expected subjects.

Info:  clusterrolebinding/system:build-strategy-custom-binding is missing subject {SystemGroup  system:authenticated    }.

WARN:  [CRBD1003 from diagnostic ClusterRoleBindings@openshift/origin/pkg/diagnostics/cluster/rolebindings.go:83]
       clusterrolebinding/system:build-strategy-source-binding is missing expected subjects.

       Use the `oadm policy reconcile-cluster-role-bindings` command to update the role binding to include expected subjects.

Info:  clusterrolebinding/system:build-strategy-source-binding is missing subject {SystemGroup  system:authenticated    }.
```

# Hawkular Metrics #

It used to be that you had to know the metrics URL and visit https://metrics.os.example.com and accept the cert so you could view the resource usage of your pods. There is now a nice link to this URL from the pod metrics page. However, even though I can vist the URL sucessfully, the metrics tab lists a _Forbidden_ error at the bottom, and no graphs. I suspect this may be related to the metrics running `openshift3/metrics-hawkular-metrics:3.1.1` still.

I could just dump the stats and [start over](https://docs.openshift.com/enterprise/3.2/install_config/cluster_metrics.html), but I'll try updating the replication controllers first.

Get the list, and then use `oc edit` to set the version to 3.2.0.

```bash
$ oc get rc -n openshift-infra -o wide
NAME                   DESIRED   CURRENT   AGE       CONTAINER(S)           IMAGE(S)                                                               SELECTOR
hawkular-cassandra-1   1         1         73d       hawkular-cassandra-1   registry.access.redhat.com/openshift3/metrics-cassandra:3.1.1          name=hawkular-cassandra-1
hawkular-metrics       1         1         73d       hawkular-metrics       registry.access.redhat.com/openshift3/metrics-hawkular-metrics:3.1.1   name=hawkular-metrics
heapster               1         1         73d       heapster               registry.access.redhat.com/openshift3/metrics-heapster:3.1.1           name=heapster
```

Edit the replication controller and change the image tag from `3.1.1` to `3.2.0`

```bash
$ oc edit rc hawkular-cassandra-1
```

The pod needs to be removed so the replication controller can recreate it. I think you could pre-emptively pull the new image if you are worried about the downtime.

```bash
$ oc get pods
NAME                         READY     STATUS    RESTARTS   AGE
hawkular-cassandra-1-t7mp5   1/1       Running   0          4d
hawkular-metrics-2ccne       1/1       Running   0          4d
heapster-6usys               1/1       Running   4          4d

$ oc delete pod hawkular-cassandra-1-t7mp5
pod "hawkular-cassandra-1-t7mp5" deleted

$ oc get events --watch
```

Unfortunately I wound up with a `CrashLoopBackOff` for the cassandra pod. Watching the logs it seemed to be replaying transactions, so I'm not sure if it was a timeout or what exactly.

```
2016-05-20 21:03:15 -0700 PDT   2016-05-20 21:18:03 -0700 PDT   61        hawkular-cassandra-1-iwl9z   Pod       spec.containers{hawkular-cassandra-1}   Warning   BackOff   {kubelet ose-prod-node-06.example.com}   Back-off restarting failed docker container
2016-05-20 21:11:42 -0700 PDT   2016-05-20 21:18:03 -0700 PDT   30        hawkular-cassandra-1-iwl9z   Pod                 Warning   FailedSync   {kubelet ose-prod-node-06.example.com}   Error syncing pod, skipping: failed to "StartContainer" for "hawkular-cassandra-1" with CrashLoopBackOff: "Back-off 5m0s restarting failed container=hawkular-cassandra-1 pod=hawkular-cassandra-1-iwl9z_openshift-infra(ac1669d8-1f08-11e6-8f0c-001a4a48be57)"
```

```
$ oc logs hawkular-cassandra-1-iwl9z -f
About to generate seeds
Trying to access the Seed list [try #1]
Trying to access the Seed list [try #2]
Trying to access the Seed list [try #3]
Setting seeds to be hawkular-cassandra-1-iwl9z
cat: /etc/ld.so.conf.d/*.conf: No such file or directory
CompilerOracle: inline org/apache/cassandra/db/AbstractNativeCell.compareTo (Lorg/apache/cassandra/db/composites/Composite;)I
CompilerOracle: inline org/apache/cassandra/db/composites/AbstractSimpleCellNameType.compareUnsigned (Lorg/apache/cassandra/db/composites/Composite;Lorg/apache/cassandra/db/composites/Composite;)I
CompilerOracle: inline org/apache/cassandra/io/util/Memory.checkBounds (JJ)V
CompilerOracle: inline org/apache/cassandra/io/util/SafeMemory.checkBounds (JJ)V
CompilerOracle: inline org/apache/cassandra/utils/AsymmetricOrdering.selectBoundary (Lorg/apache/cassandra/utils/AsymmetricOrdering/Op;II)I
CompilerOracle: inline org/apache/cassandra/utils/AsymmetricOrdering.strictnessOfLessThan (Lorg/apache/cassandra/utils/AsymmetricOrdering/Op;)I
CompilerOracle: inline org/apache/cassandra/utils/ByteBufferUtil.compare (Ljava/nio/ByteBuffer;[B)I
CompilerOracle: inline org/apache/cassandra/utils/ByteBufferUtil.compare ([BLjava/nio/ByteBuffer;)I
CompilerOracle: inline org/apache/cassandra/utils/ByteBufferUtil.compareUnsigned (Ljava/nio/ByteBuffer;Ljava/nio/ByteBuffer;)I
CompilerOracle: inline org/apache/cassandra/utils/FastByteOperations$UnsafeOperations.compareTo (Ljava/lang/Object;JILjava/lang/Object;JI)I
CompilerOracle: inline org/apache/cassandra/utils/FastByteOperations$UnsafeOperations.compareTo (Ljava/lang/Object;JILjava/nio/ByteBuffer;)I
CompilerOracle: inline org/apache/cassandra/utils/FastByteOperations$UnsafeOperations.compareTo (Ljava/nio/ByteBuffer;Ljava/nio/ByteBuffer;)I
INFO  04:07:55 Loading settings from file:/opt/apache-cassandra-2.2.1.redhat-2/conf/cassandra.yaml
INFO  04:07:55 Node configuration:[authenticator=AllowAllAuthenticator; authorizer=AllowAllAuthorizer; auto_snapshot=true; batch_size_warn_threshold_in_kb=5; batchlog_replay_throttle_in_kb=1024; cas_conte
ntion_timeout_in_ms=1000; client_encryption_options=<REDACTED>; cluster_name=hawkular-metrics; column_index_size_in_kb=64; commit_failure_policy=stop; commitlog_directory=/cassandra_data/commitlog; commit
log_segment_size_in_mb=32; commitlog_sync=periodic; commitlog_sync_period_in_ms=10000; compaction_throughput_mb_per_sec=16; concurrent_counter_writes=32; concurrent_reads=32; concurrent_writes=32; counter
_cache_save_period=7200; counter_cache_size_in_mb=null; counter_write_request_timeout_in_ms=5000; cross_node_timeout=false; data_file_directories=[/cassandra_data/data]; disk_failure_policy=stop; dynamic_
snitch_badness_threshold=0.1; dynamic_snitch_reset_interval_in_ms=600000; dynamic_snitch_update_interval_in_ms=100; endpoint_snitch=SimpleSnitch; hinted_handoff_enabled=true; hinted_handoff_throttle_in_kb
=1024; incremental_backups=false; index_summary_capacity_in_mb=null; index_summary_resize_interval_in_minutes=60; inter_dc_tcp_nodelay=false; internode_compression=all; key_cache_save_period=14400; key_ca
che_size_in_mb=null; listen_address=hawkular-cassandra-1-iwl9z; max_hint_window_in_ms=10800000; max_hints_delivery_threads=2; memtable_allocation_type=heap_buffers; native_transport_port=9042; num_tokens=
256; partitioner=org.apache.cassandra.dht.Murmur3Partitioner; permissions_validity_in_ms=2000; range_request_timeout_in_ms=10000; read_request_timeout_in_ms=5000; request_scheduler=org.apache.cassandra.sc
heduler.NoScheduler; request_timeout_in_ms=10000; row_cache_save_period=0; row_cache_size_in_mb=0; rpc_address=hawkular-cassandra-1-iwl9z; rpc_keepalive=true; rpc_port=9160; rpc_server_type=sync; seed_pro
vider=[{class_name=org.hawkular.openshift.cassandra.OpenshiftSeedProvider, parameters=[{seeds=hawkular-cassandra-1-iwl9z}]}]; server_encryption_options=<REDACTED>; snapshot_before_compaction=false; ssl_st
orage_port=7001; sstable_preemptive_open_interval_in_mb=50; start_native_transport=true; start_rpc=true; storage_port=7000; thrift_framed_transport_size_in_mb=15; tombstone_failure_threshold=100000; tombs
tone_warn_threshold=1000; trickle_fsync=false; trickle_fsync_interval_in_kb=10240; truncate_request_timeout_in_ms=60000; write_request_timeout_in_ms=2000]
INFO  04:07:55 DiskAccessMode 'auto' determined to be mmap, indexAccessMode is mmap
INFO  04:07:56 Global memtable on-heap threshold is enabled at 125MB
INFO  04:07:56 Global memtable off-heap threshold is enabled at 125MB
WARN  04:07:56 UnknownHostException for service 'hawkular-cassandra-nodes'. It may not be up yet. Trying again
WARN  04:07:58 UnknownHostException for service 'hawkular-cassandra-nodes'. It may not be up yet. Trying again
WARN  04:08:00 UnknownHostException for service 'hawkular-cassandra-nodes'. It may not be up yet. Trying again
WARN  04:08:02 UnknownHostException for service 'hawkular-cassandra-nodes'. It may not be up yet. Trying again
...
```

Not only that, but it looks like there are some compatibility issues.

```bash
$ oc logs heapster-jdpgi
exec: "./heapster-wrapper.sh": stat ./heapster-wrapper.sh: no such file or directory
```

There is a new debug command in 3.2, but I wasn't able to learn much from it other than the startup command for the pod.

```bash
$ oc debug pod heapster-jdpgi
Debugging with pod/heapster-jdpgi-debug, original command: ./heapster-wrapper.sh --wrapper.username_file=/hawkular-account/hawkular-metrics.username --wrapper.password_file=/hawkular-account/hawkular-metrics.password --wrapper.allowed_users_file=/secrets/heapster.allowed-users --source=kubernetes:https://kubernetes.default.svc:443?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250 --sink=hawkular:https://hawkular-metrics:443?tenant=_system&labelToTenant=pod_namespace&caCert=/hawkular-cert/hawkular-metrics-ca.certificate&user=%username%&pass=%password%&filter=label(container_name:^/system.slice.*|^/user.slice) --logtostderr=true --tls_cert=/secrets/heapster.cert --tls_key=/secrets/heapster.key --tls_client_ca=/secrets/heapster.client-ca --allowed_users=%allowed_users%
```

## Redeploy Cluster Metrics ##

So, it seems there is some missing documentation here.  I'm going to go ahead and [redeploy cluster metrics](https://docs.openshift.com/enterprise/3.2/install_config/cluster_metrics.html) instead of trying to fix it.

- Clear the decks and remove the old deployment

```bash
$ oc delete all --selector="metrics-infra"
replicationcontroller "hawkular-cassandra-1" deleted
replicationcontroller "hawkular-metrics" deleted
replicationcontroller "heapster" deleted
route "hawkular-metrics" deleted
service "hawkular-cassandra" deleted
service "hawkular-cassandra-nodes" deleted
service "hawkular-metrics" deleted
service "heapster" deleted
pod "hawkular-cassandra-1-iwl9z" deleted

$ oc delete templates --selector="metrics-infra"
template "hawkular-cassandra-node-emptydir" deleted
template "hawkular-cassandra-node-pv" deleted
template "hawkular-cassandra-services" deleted
template "hawkular-heapster" deleted
template "hawkular-metrics" deleted
template "hawkular-support" deleted

$ oc delete secrets --selector="metrics-infra"
secret "hawkular-cassandra-certificate" deleted
secret "hawkular-cassandra-secrets" deleted
secret "hawkular-metrics-account" deleted
secret "hawkular-metrics-certificate" deleted
secret "hawkular-metrics-secrets" deleted
secret "heapster-secrets" deleted

$ oc delete pvc --selector="metrics-infra"
persistentvolumeclaim "metrics-cassandra-1" deleted

$ oc delete sa --selector="metrics-infra"
serviceaccount "cassandra" deleted
serviceaccount "hawkular" deleted
serviceaccount "heapster" deleted
```

The _Reclaim Policy_ on the PV used by metrics is _retain_, so go empty out that volume before continuing.

```bash
$ oc describe pv metrics
Name:           metrics
Labels:         <none>
Status:         Released
Claim:          openshift-infra/metrics-cassandra-1
Reclaim Policy: Retain
Access Modes:   RWO,RWX
Capacity:       50Gi
Message:
Source:
    Type:       NFS (an NFS mount that lasts the lifetime of a pod)
    Server:     openshift-data.example.com
    Path:       /openshift/prod/metrics
    ReadOnly:   false
```

- Deploy the metrics again and wait for it to complete.

```bash
$ oc process -f /usr/share/ansible/openshift-ansible/roles/openshift_examples/files/examples/v1.2/infrastructure-templates/enterprise/metrics-deployer.yaml \
  -v HAWKULAR_METRICS_HOSTNAME=metrics.os.example.com \
  | oc create -f -
```

- Now replace the hawkular route with a [reencrypting router]( https://docs.openshift.com/enterprise/latest/install_config/cluster_metrics.html#metrics-reencrypting-route)

```bash
$ oc describe route hawkular-metrics
Name:                   hawkular-metrics
Created:                2 hours ago
Labels:                 metrics-infra=support
Annotations:            <none>
Requested Host:         metrics.os.example.com
                          exposed on router ha-router-primary 2 hours ago
Path:                   <none>
TLS Termination:        passthrough
Insecure Policy:        <none>
Service:                hawkular-metrics
Endpoint Port:          <all endpoint ports>
Endpoints:              10.1.7.5:8444

$ oc describe service hawkular-metrics
Name:                   hawkular-metrics
Namespace:              openshift-infra
Labels:                 metrics-infra=hawkular-metrics,name=hawkular-metrics
Selector:               name=hawkular-metrics
Type:                   ClusterIP
IP:                     172.30.113.214
Port:                   https-endpoint  443/TCP
Endpoints:              10.1.7.5:8444
Session Affinity:       None
No events.
```

- Create `route-metrics-reencrypt.yaml` with the _wildcard.os.example.com_ cert.

```bash
$ oc get routes
NAME               HOST/PORT              PATH      SERVICE            TERMINATION   LABELS
hawkular-metrics   metrics.os.example.com             hawkular-metrics   passthrough   metrics-infra=support

$ vim route-metrics-reencrypt.yaml

$ oc delete route hawkular-metrics
route "hawkular-metrics" deleted

$ oc create -f route-metrics-reencrypt.yaml
route "hawkular-metrics" created

$ oc get routes
NAME               HOST/PORT              PATH      SERVICE                 TERMINATION   LABELS
hawkular-metrics   metrics.os.example.com             hawkular-metrics:8444   reencrypt     metrics-infra=support
```

After this, metrics work again, even better! FWIW there is some mixed content on the page.

> Mixed Content: The page at 'https://metrics.os.example.com/hawkular/metrics' was loaded over HTTPS, but requested an insecure stylesheet 'http://fonts.googleapis.com/css?family=Exo+2'. This request has been blocked; the content must be served over HTTPS.
