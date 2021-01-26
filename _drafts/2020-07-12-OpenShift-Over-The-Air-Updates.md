---
title: OpenShift Operator Driven Over The Air Updates
layout: post
tags:
 - openshift
 - OCP4
---


OpenShift 4.x extends the [operator pattern](https://github.com/operator-framework/) to enable self-management of the cluster services and underlying resources leveraging over-the-air updates much like you are accustomed to receiving for your smart phone.

**What is an operator?**

[An Operator is](https://docs.openshift.com/container-platform/latest/operators/olm-what-operators-are.html) a method of packaging, deploying, and managing a Kubernetes application.

**What is a "kubernetes application"?**

[A Kubernetes application is](https://docs.openshift.com/container-platform/latest/operators/olm-what-operators-are.html) an app that is both deployed on Kubernetes and managed using the Kubernetes APIs.

OpenShift enables users or administrators to install third-party applications from catalog sources such as [OperatorHub](https://operatorhub.io/) and to [build and bundle their own operators](https://github.com/operator-framework/operator-sdk) that can be made available via a custom catalog source. The lifecycle of operators such as these is managed by the [Operator Lifecycle Manager](https://github.com/operator-framework/operator-lifecycle-manager) or OLM.

There is another [set of cluster operators](https://docs.openshift.com/container-platform/latest/operators/operator-reference.html) that are not optional, but make up the services underpinning the OpenShift cluster. These are called Cluster Operators and they are managed by the [Cluster Version Operator](https://github.com/openshift/cluster-version-operator) or CVO.

The CVO is what is responsible for orchestrating what we call over the air updates. By consuming a payload representing the artifacts making up an OpenShift release, the CVO can ensure that everything from the RHEL CoreOS version on the nodes to the OpenShift web console are in sync.

For more information on this topic including a video demonstration of the process of upgrading OpenShift on the command line or via the web browser, please see [my BrightTALK presentation](https://www.brighttalk.com/webcast/14777/433967) from September, 2020.


[![OpenShift Over-The-Air Updates BrightTALK](/images/openshift-ota-brighttalk.png)](https://www.brighttalk.com/webcast/14777/433967)

