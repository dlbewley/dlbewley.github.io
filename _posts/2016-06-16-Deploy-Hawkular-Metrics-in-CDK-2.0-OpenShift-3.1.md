---
title: Deploy Hawkular Metrics in CDK 2.1 OpenShift 3.2
layout: post
tags:
 - kubernetes
 - openshift
 - OSE3.2
 - CDK2.1
---

**Update!** _I failed with CDK 2.0, but CDK 2.1 works with some fiddling._

In my [last post](http://guifreelife.com/blog/2016/06/16/Getting-Started-With-RedHat-Container-Development-Kit) I installed Red Hat Container Developer Kit to deploy OpenShift Enterprise using Vagrant. But now I want to add Hawkular Metrics to that deployment.

# Deploy Metrics #

Refer to [the docs](https://docs.openshift.com/enterprise/3.2/install_config/cluster_metrics.html) for deploying metrics in OSE.

[![OpenShift Metrics](/images/thumb/openshift-cdk-metrics-0.png)](/images/openshift-cdk-metrics-0.png)

Login to the vagrant CDK VM before continuing

```bash
$ cd ~/cdk/components/rhel/rhel-ose/
$ vagrant ssh

$ oc login
Authentication required for https://127.0.0.1:8443 (openshift)
Username: admin
Password: admin
Login successful.

$ oc project openshift-infra

$ oc get sa
NAME                        SECRETS   AGE
build-controller            2         10m
builder                     2         10m
daemonset-controller        2         10m
default                     2         10m
deployer                    2         10m
deployment-controller       2         10m
gc-controller               2         10m
hpa-controller              2         10m
job-controller              2         10m
namespace-controller        2         10m
pv-binder-controller        2         10m
pv-provisioner-controller   2         10m
pv-recycler-controller      2         10m
replication-controller      2         10m

$ oc create -f - <<API
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-deployer
  secrets:
  - name: metrics-deployer
API

$ oc secrets new metrics-deployer nothing=/dev/null

$ oadm policy add-role-to-user         edit           system:serviceaccount:openshift-infra:metrics-deployer
$ oadm policy add-cluster-role-to-user cluster-reader system:serviceaccount:openshift-infra:heapster
```

From your OSE server grab  `/usr/share/openshift/examples/infrastructure-templates/enterprise/metrics-deployer.yaml` or from [here](https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v1.2/infrastructure-templates/enterprise/metrics-deployer.yaml)

```bash
$ curl -O https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v1.2/infrastructure-templates/enterprise/metrics-deployer.yaml

$ oc process -f metrics-deployer.yaml \
             -v HAWKULAR_METRICS_HOSTNAME=metrics.10.1.2.2.xip.io \
             -v USE_PERSISTENT_STORAGE=false \
             | oc create -f -

# be patient while the images are pulled and pods are started. this can take a long time.
$ oc get events --watch
```

You are probably doing to see [cassandra errors](#cassandra-errors), but continue to the next step to [tell the master](#update-openshift-master-config) where to find the metrics. 

## Cassandra Errors ##

The events may eventually output something like this:

```
2016-07-04 21:10:03 -0400 EDT   2016-07-04 21:10:03 -0400 EDT   1         hawkular-cassandra-1-5kmz4   Pod       spec.containers{hawkular-cassandra-1}   Warning   Unhealthy   {kubelet rhel-cdk}   Readiness probe failed: cat: /etc/ld.so.conf.d/*.conf: No such file or directory
nodetool: Failed to connect to '127.0.0.1:7199' - ConnectException: 'Connection refused'.
Cassandra not in the up and normal state. Current state is
/opt/apache-cassandra/bin/cassandra-docker-ready.sh: line 28: [: =: unary operator expected
```

Output a little more info on that pod:

```
[vagrant@rhel-cdk ~]$ oc describe pod hawkular-cassandra-1-5kmz4
Name:           hawkular-cassandra-1-5kmz4
Namespace:      openshift-infra
Node:           rhel-cdk/10.0.2.15
Start Time:     Mon, 04 Jul 2016 20:56:41 -0400
Labels:         metrics-infra=hawkular-cassandra,name=hawkular-cassandra-1,type=hawkular-cassandra
Status:         Running
IP:             172.17.0.2
Controllers:    ReplicationController/hawkular-cassandra-1
Containers:
  hawkular-cassandra-1:
    Container ID:       docker://a5716703d9b98a540255e3db8cb40f15b39127e47f5e1f5279a8f63e07e47903
    Image:              registry.access.redhat.com/openshift3/metrics-cassandra:3.2.1
    Image ID:           docker://2a27e048703696117de74856c629f6837399e621658ece4fc725e4fe8c54bbcd
    Ports:              9042/TCP, 9160/TCP, 7000/TCP, 7001/TCP
    Command:
      /opt/apache-cassandra/bin/cassandra-docker.sh
      --cluster_name=hawkular-metrics
      --data_volume=/cassandra_data
      --internode_encryption=all
      --require_node_auth=true
      --enable_client_encryption=true
      --require_client_auth=true
      --keystore_file=/secret/cassandra.keystore
      --keystore_password_file=/secret/cassandra.keystore.password
      --truststore_file=/secret/cassandra.truststore
      --truststore_password_file=/secret/cassandra.truststore.password
      --cassandra_pem_file=/secret/cassandra.pem
    QoS Tier:
      memory:           BestEffort
      cpu:              BestEffort
    State:              Running
      Started:          Mon, 04 Jul 2016 21:26:17 -0400
    Ready:              True
    Restart Count:      0
    Readiness:          exec [/opt/apache-cassandra/bin/cassandra-docker-ready.sh] delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment Variables:
      CASSANDRA_MASTER: true
      POD_NAMESPACE:    openshift-infra (v1:metadata.namespace)
Conditions:
  Type          Status
  Ready         True
Volumes:
  cassandra-data:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  hawkular-cassandra-secrets:
    Type:       Secret (a volume populated by a Secret)
    SecretName: hawkular-cassandra-secrets
  cassandra-token-a2muv:
    Type:       Secret (a volume populated by a Secret)
    SecretName: cassandra-token-a2muv
Events:
  FirstSeen     LastSeen        Count   From                    SubobjectPath                           Type            Reason          Message
  ---------     --------        -----   ----                    -------------                           --------        ------          -------
  32m           32m             1       {default-scheduler }                                            Normal          Scheduled       Successfully assigned hawkular-cassandra-1-5kmz4 to rhel-cdk
  32m           32m             1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal          Pulling         pulling image "registry.access.redhat.com/openshift3/metrics-c
assandra:3.2.1"
  19m           19m             1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal          Pulled          Successfully pulled image "registry.access.redhat.com/openshif
t3/metrics-cassandra:3.2.1"
  19m           19m             1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal          Created         Created container with docker id 06fd27c4047c
  19m           19m             1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal          Started         Started container with docker id 06fd27c4047c
  19m           19m             1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Warning         Unhealthy       Readiness probe failed: cat: /etc/ld.so.conf.d/*.conf: No such
 file or directory
nodetool: Failed to connect to '127.0.0.1:7199' - ConnectException: 'Connection refused'.
Cassandra not in the up and normal state. Current state is
/opt/apache-cassandra/bin/cassandra-docker-ready.sh: line 28: [: =: unary operator expected

  2m    2m      1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal  Pulled          Container image "registry.access.redhat.com/openshift3/metrics-cassandra:3.2.1" alread
y present on machine
  2m    2m      1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal  Created         Created container with docker id a5716703d9b9
  2m    2m      1       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Normal  Started         Started container with docker id a5716703d9b9
  2m    2m      2       {kubelet rhel-cdk}      spec.containers{hawkular-cassandra-1}   Warning Unhealthy       Readiness probe failed: cat: /etc/ld.so.conf.d/*.conf: No such file or directory
nodetool: Failed to connect to '127.0.0.1:7199' - ConnectException: 'Connection refused'.
Cassandra not in the up and normal state. Current state is
/opt/apache-cassandra/bin/cassandra-docker-ready.sh: line 28: [: =: unary operator expected
```

There is a problem with the readiness check of the cassandra pod. Using [this commit](https://github.com/openshift/origin-metrics/commit/cf2c6d0088426e285178a9e3057156e9a90dc56f#diff-c13869165b2db409fcc3d8d14f188a32) make a change to the running pod. Basically, on line 28, change `$STATUS` to `${STATUS}`.

Now, even though there is the error catting a non-existant file, the script will not error out:

```bash
oc rsh hawkular-cassandra-1-5kmz4
sh-4.2$ /opt/apache-cassandra/bin/cassandra-docker-ready.sh
cat: /etc/ld.so.conf.d/*.conf: No such file or directory
Cassandra is in the up and normal state. It is now ready
```

# Update OpenShift Master Config #

Openshift its self is running in a container called `openshift`. The config dir is mounted from the CDK VM at `/var/lib/openshift/openshift.local.config/master`

```bash
sudo vi /var/lib/openshift/openshift.local.config/master/master-config.yaml

# Add this:
assetConfig:
  masterPublicURL: https://10.1.2.2:8443
  metricsPublicURL: "https://metrics.10.1.2.2.xip.io/hawkular/metrics"
```

Login to the openshift container and HUP it. _There is probably a better way to do this. In CDK 2.1 this killed OpenShift. Instead do a `sudo systemctl restart openshift` from the VM_ 

```bash
# from host
vagrant ssh
# from VM
docker exec -ti openshift bash
ps -ef | grep openshift
kill -HUP <pid of openshift container>
exit
```

# Finish Up #

Visit [https://metrics.10.1.2.2.xip.io/hawkular/metrics](https://metrics.10.1.2.2.xip.io/hawkular/metrics) to confirm it is running, and accept the SSL certificate.

Again, be patient. There are several docker pulls going on which take quite some time.

# Do Over #

If you need to just clear the decks and start over, do this then go back to top.

```
oc delete all --selector="metrics-infra"
oc delete templates --selector="metrics-infra"
oc delete secrets --selector="metrics-infra"
oc delete pvc --selector="metrics-infra"
oc delete sa --selector="metrics-infra"

oc secrets new metrics-deployer nothing=/dev/null
```

