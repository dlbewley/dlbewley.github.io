---
title: OpenShift Cluster Metrics and Cassandra Troubleshooting
layout: post
tags:
 - cassandra
 - hawkular
 - kubernetes
 - openshift
 - OCP3.3
 - metrics
---

OpenShift gathers [cluster metrics](https://docs.openshift.com/container-platform/3.3/install_config/cluster_metrics.html) such as CPU, memory, and network bandwidth per pod which can assist in troubleshooting and capacity planning. The metrics are also used to support [horizontal pod autoscaling](https://docs.openshift.com/container-platform/3.3/dev_guide/pod_autoscaling.html), which makes the metrics service not just helpful, but critical to operation.

## Missing Liveness Probes ##

There are 3 major components in the metrics collection process. [Heapster](https://github.com/kubernetes/heapster) gathers stats from Docker and feeds them to [Hawkular Metrics](https://github.com/hawkular/hawkular-metrics) to tuck away for safe keeping in [Cassandra](http://cassandra.apache.org/).

To ensure the services are healthy [readiness and liveness probes](https://docs.openshift.com/container-platform/3.3/dev_guide/application_health.html) are deployed. The following checks are created on the replicationControllers by the metrics deployer:

- _heapster_ 
  - `readinessprobe`: `/opt/heapster-readiness.sh`
  - `livenessprobe`: _None_

- _hawkular-metrics_
  - `readinessprobe`: `/opt/hawkular/scripts/hawkular-metrics-readiness.py`
  - `livenessprobe`: `/opt/hawkular/scripts/hawkular-metrics-liveness.py`

- _hawkular-cassandra_
  - `readinessprobe`: `/opt/apache-cassandra/bin/cassandra-docker-ready.sh`
  - `livenessprobe`: _None_

**When is OK not OK?**

I have seen things in a state such that _heapster_ is _Ready_, but logging:

> Failed to push data to sink: Hawkular-Metrics Sink

At the same time _hawkular-metrics_ purports to be ready but is logging:

> Caused by: com.datastax.driver.core.exceptions.WriteTimeoutException: Cassandra timeout during write query at consistency LOCAL_ONE (1 replica were required but only 0 acknowledged the write)

And concurrent to that, _hawkular-cassandra_ is supposedly _OK_ per `nodeutil status` as called from `/opt/apache-cassandra/bin/cassandra-docker-ready.sh`.

> Caused by: com.datastax.driver.core.exceptions.WriteTimeoutException: Cassandra timeout during write query at consistency LOCAL_ONE (1 replica were required but only 0 acknowledged the write)

Unfortunately there is no liveness probe defined for Cassandra, so if it fails your metrics collection fails,  and it will not self-heal. Additionally you likely will not know unless your developers complain about missing the metrics graphs on their pods.

I was informed by Red Hat Support that running `nodetool tablestats hawkular-metrics` periodically can prove that metric record count is increasing. This might be adequate for a cassandra liveness probe, but I have not yet explored this.

## Cassandra Heap OOM ##

Today I noticed my metrics were malfunctioning and I observed that cassandra is out of heap space:

```bash
$ oc logs hawkular-cassandra-1-eo3w8 | grep 'heap space'
...
java.lang.OutOfMemoryError: Java heap space
```

**How big is the cassandra heap space?##

First off, how does the cassandra container start?

```bash
$ oc get rc hawkular-cassandra-1 -o json | jq .spec.template.spec.containers[].command
[
  "/opt/apache-cassandra/bin/cassandra-docker.sh",
  "--cluster_name=hawkular-metrics",
  "--data_volume=/cassandra_data",
  "--internode_encryption=all",
  "--require_node_auth=true",
  "--enable_client_encryption=true",
  "--require_client_auth=true",
  "--keystore_file=/secret/cassandra.keystore",
  "--keystore_password_file=/secret/cassandra.keystore.password",
  "--truststore_file=/secret/cassandra.truststore",
  "--truststore_password_file=/secret/cassandra.truststore.password",
  "--cassandra_pem_file=/secret/cassandra.pem"
]
```

I am using v3.3.0 of the image:

```
$ oc get rc hawkular-cassandra-1 -o json | jq .spec.template.spec.containers[].image
"registry.access.redhat.com/openshift3/metrics-cassandra:3.3.0"
```

Looking at the startup `/opt/apache-cassandra/bin/cassandra-docker.sh` script in the pod there is this:

```bash
if [ -z "${MAX_HEAP_SIZE}" ]; then
  if [ -z "${MEMORY_LIMIT}" ]; then
    MEMORY_LIMIT=`cat /sys/fs/cgroup/memory/memory.limit_in_bytes`
    echo "The MEMORY_LIMIT envar was not set. Reading value from /sys/fs/cgroup/memory/memory.limit_in_bytes."
  fi
  echo "The MAX_HEAP_SIZE envar is not set. Basing the MAX_HEAP_SIZE on the available memory limit for the pod (${MEMORY_LIMIT})."
  BYTES_MEGABYTE=1048576
  BYTES_GIGABYTE=1073741824
  # Based on the Cassandra memory limit recommendations. See http://docs.datastax.com/en/cassandra/2.2/cassandra/operations/opsTuneJVM.html
  if (( ${MEMORY_LIMIT} <= (2 * ${BYTES_GIGABYTE}) )); then
    # If less than 2GB, set the heap to be 1/2 of available ram
    echo "The memory limit is less than 2GB. Using 1/2 of available memory for the max_heap_size."
    export MAX_HEAP_SIZE="$((${MEMORY_LIMIT} / ${BYTES_MEGABYTE} / 2 ))M"
  elif (( ${MEMORY_LIMIT} <= (4 * ${BYTES_GIGABYTE}) )); then
    echo "The memory limit is between 2 and 4GB. Setting max_heap_size to 1GB."
    # If between 2 and 4GB, set the heap to 1GB
    export MAX_HEAP_SIZE="1024M"
  elif (( ${MEMORY_LIMIT} <= (32 * ${BYTES_GIGABYTE}) )); then
    echo "The memory limit is between 4 and 32GB. Using 1/4 of the available memory for the max_heap_size."
    # If between 4 and 32GB, use 1/4 of the available ram
    export MAX_HEAP_SIZE="$(( ${MEMORY_LIMIT} / ${BYTES_MEGABYTE} / 4 ))M"
  else
    echo "The memory limit is above 32GB. Using 8GB for the max_heap_size"
    # If above 32GB, set the heap size to 8GB
    export MAX_HEAP_SIZE="8192M"
  fi
  echo "The MAX_HEAP_SIZE has been set to ${MAX_HEAP_SIZE}"
else
  echo "The MAX_HEAP_SIZE envar is set to ${MAX_HEAP_SIZE}. Using this value"
fi

if [ -z "${HEAP_NEWSIZE}" ] && [ -z "${CPU_LIMIT}" ]; then
  echo "The HEAP_NEWSIZE and CPU_LIMIT envars are not set. Defaulting the HEAP_NEWSIZE to 100M"
  export HEAP_NEWSIZE=100M
elif [ -z "${HEAP_NEWSIZE}" ]; then
  export HEAP_NEWSIZE=$((CPU_LIMIT/10))M
  echo "THE HEAP_NEWSIZE envar is not set. Setting to ${HEAP_NEWSIZE} based on the CPU_LIMIT of ${CPU_LIMIT}. [100M per CPU core]"
else
  echo "The HEAP_NEWSIZE envar is set to ${HEAP_NEWSIZE}. Using this value"
fi
```

So using `oc rsh` to examine the pod a little more.

```bash
$ oc rsh hawkular-cassandra-1-eo3w8 env | grep MAX
$ oc rsh hawkular-cassandra-1-eo3w8 env | grep MEMORY
MEMORY_LIMIT=202622070784
$ oc rsh hawkular-cassandra-1-eo3w8 cat /sys/fs/cgroup/memory/memory.limit_in_bytes
9223372036854775807
```

At start up the container outputs the following.

> The MAX_HEAP_SIZE envar is not set. Basing the MAX_HEAP_SIZE on the available memory limit for the pod (202622070784).
> The memory limit is above 32GB. Using 8GB for the max_heap_size
> The MAX_HEAP_SIZE has been set to 8192M
> THE HEAP_NEWSIZE envar is not set. Setting to 3200M based on the CPU_LIMIT of 32000. [100M per CPU core]

All this tells me Cassandra's Java heap is being artificially limited to 8G (on my node which has 192G RAM), and to fix that I can set the `MAX_HEAP_SIZE` variable in the replication controller config.

**Before:**

```bash
$ oc get rc hawkular-cassandra-1 -o json | jq .spec.template.spec.containers[].env
```
```json
[
  {
    "name": "CASSANDRA_MASTER",
    "value": "true"
  },
  {
    "name": "POD_NAMESPACE",
    "valueFrom": {
      "fieldRef": {
        "apiVersion": "v1",
        "fieldPath": "metadata.namespace"
      }
    }
  },
  {
    "name": "MEMORY_LIMIT",
    "valueFrom": {}
  },
  {
    "name": "CPU_LIMIT",
    "valueFrom": {}
  }
]
```

**TODO**

- What is a good value? [ref1](https://docs.datastax.com/en/cassandra/2.1/cassandra/operations/ops_tune_jvm_c.html), [ref2](http://stackoverflow.com/questions/30207779/optimal-jvm-settings-for-cassandra), [ref3](https://tobert.github.io/pages/als-cassandra-21-tuning-guide.html)
  - It doesn't look like JMX is enabled since Sysdig is not showing me any detail about the heap usage. I think I will increase it to 12GB without having done complete due diligence.

- Can this be [injected via the template](https://docs.openshift.com/container-platform/3.3/install_config/cluster_metrics.html#deployer-template-parameters) at deploy time?
  - Nope. I don't see a param in `/usr/share/openshift/examples/infrastructure-templates/enterprise/metrics-deployer.yaml`