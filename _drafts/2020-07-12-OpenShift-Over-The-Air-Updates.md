---
title: OpenShift Operator Driven Over The Air Updates
layout: post
tags:
 - openshift
 - OCP4
---


OpenShift 4.x extends the [operator pattern](https://github.com/operator-framework/) to enable self-management of the cluster services and underlying server resources to enable over-the-air updates much like you are accustomed to receiving for your smart phone.

**What is an operator?**

[An Operator is](https://docs.openshift.com/container-platform/latest/operators/olm-what-operators-are.html) a method of packaging, deploying, and managing a _Kubernetes application_.

**What is a "kubernetes application"?**

[A Kubernetes application is](https://docs.openshift.com/container-platform/latest/operators/olm-what-operators-are.html) an app that is both deployed on Kubernetes and managed using the Kubernetes APIs.

OpenShift enables users or administrators to install third-party applications from catalog sources such as [OperatorHub](https://operatorhub.io/) and to [build and bundle their own operators](https://github.com/operator-framework/operator-sdk) which can be made available for installation via a custom catalog source. After installation, the lifecycle of these operators is managed through a subscription by the [Operator Lifecycle Manager](https://github.com/operator-framework/operator-lifecycle-manager) or OLM.

There is another [set of cluster operators](https://docs.openshift.com/container-platform/latest/operators/operator-reference.html) that do not require manual installation. These operators make up the services underpinning the OpenShift cluster. These are called Cluster Operators, and you can see them with the `oc get clusteroperators command`.

{% highlight shell %}
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.6.8     True        False         False      26m
cloud-credential                           4.6.8     True        False         False      88d
cluster-autoscaler                         4.6.8     True        False         False      88d
config-operator                            4.6.8     True        False         False      88d
console                                    4.6.8     True        False         False      9m4s
csi-snapshot-controller                    4.6.8     True        False         False      26m
dns                                        4.6.4     True        False         False      75d
etcd                                       4.6.8     True        False         False      88d
image-registry                             4.6.8     True        False         False      68d
ingress                                    4.6.8     True        False         False      68d
insights                                   4.6.8     True        False         False      88d
kube-apiserver                             4.6.8     True        False         False      88d
kube-controller-manager                    4.6.8     True        False         False      88d
kube-scheduler                             4.6.8     True        False         False      88d
kube-storage-version-migrator              4.6.8     True        False         False      3d14h
machine-api                                4.6.8     True        False         False      88d
machine-approver                           4.6.8     True        False         False      88d
machine-config                             4.6.4     True        False         False      29m
...truncated...
{% endhighlight %}

These cluster operators are managed by the [Cluster Version Operator](https://github.com/openshift/cluster-version-operator) or CVO.

The CVO is responsible for orchestrating what we call over the air updates. By consuming a payload representing the artifacts making up an OpenShift release, the CVO can ensure that everything from the RHEL CoreOS version on the nodes to the OpenShift web console are in sync. How does the CVO know what is a viable upgrade target?

For more information on this topic including a video demonstration of the process of upgrading OpenShift on the command line or via the web browser, please see [my BrightTALK presentation](https://www.brighttalk.com/webcast/14777/433967) from September, 2020.


[![OpenShift Over-The-Air Updates BrightTALK](/images/openshift-ota-brighttalk.png)](https://www.brighttalk.com/webcast/14777/433967)

