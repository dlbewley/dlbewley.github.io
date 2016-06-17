---
title: Deploy Hawkular Metrics in CDK 2.0 OpenShift 3.1
layout: post
tags:
 - kubernetes
 - openshift
 - OSE3.1
 - CDK2.0
---

In my [last post](http://guifreelife.com/blog/2016/06/16/Getting-Started-With-RedHat-Container-Development-Kit) I installed Red Hat Container Developer Kit to deploy OSE 3.1 using Vagrant. But now I want to add Hawkular Metrics.

- [Docs](https://docs.openshift.com/enterprise/3.2/install_config/cluster_metrics.html) for deploying metrics in OSE

Login to the vagrant VM before continuing

```bash
cd ~/cdk/components/rhel/rhel-ose/
vagrant ssh

oc project openshift-infra

oc create -f - <<API
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-deployer
  secrets:
  - name: metrics-deployer
API

oc secrets new metrics-deployer nothing=/dev/null

oadm policy add-role-to-user         edit           system:serviceaccount:openshift-infra:metrics-deployer
oadm policy add-cluster-role-to-user cluster-reader system:serviceaccount:openshift-infra:heapster
```

From your OSE server grab  `/usr/share/openshift/examples/infrastructure-templates/enterprise/metrics-deployer.yaml` or from [here](https://raw.githubusercontent.com/openshift/openshift-ansible/master/roles/openshift_examples/files/examples/v1.1/infrastructure-templates/enterprise/metrics-deployer.yaml)

```bash
oc process -f metrics-deployer.yaml \
             -v IMAGE_VERSION=3.1.1 \
             -v HAWKULAR_METRICS_HOSTNAME=metrics.10.1.2.2.xip.io \
             -v USE_PERSISTENT_STORAGE=false \
             | oc create -f -

oc get events --watch
```

If you need to startover do this then go back to top.

```
oc delete all --selector="metrics-infra"
oc delete templates --selector="metrics-infra"
oc delete secrets --selector="metrics-infra"
oc delete pvc --selector="metrics-infra"
oc delete sa --selector="metrics-infra"

oc secrets new metrics-deployer nothing=/dev/null
```

Visit [https://metrics.10.1.2.2.xip.io/hawkular/metrics](https://metrics.10.1.2.2.xip.io/hawkular/metrics) to confirm it is running, and accept the cert.

Openshift its self is running in a container called `openshift`. The config dir is mounted from the VM at `/var/lib/openshift/openshift.local.config/master`

```bash
sudo vi /var/lib/openshift/openshift.local.config/master/master-config.yaml

# Add this:
assetConfig:
  masterPublicURL: https://10.1.2.2:8443
  metricsPublicURL: "https://metrics.10.1.2.2.xip.io/hawkular/metrics"
```

Login to the openshift container and HUP it. _There is probably a better way to do this._

```bash
docker exec -ti openshift bash
ps -ef | grep openshift
kill -HUP <pid of openshift container>
exit
```

**ERROR**

Pods should start showing metrics before too long however, I am getting this error:

```
Error fetching cpu/usage for container helloflask.Range end must be strictly greater than start
```
