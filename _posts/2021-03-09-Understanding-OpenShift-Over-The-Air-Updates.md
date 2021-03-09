---
title: How do OpenShift Over The Air Updates Work?
layout: post
tags:
 - openshift
 - OCP4
 - operators
---

OpenShift 4 extends the [operator pattern introduced by CoreOS][9], and enables automated management of the Kubernetes cluster and the underlying resources including machine instances and operating system configuration. Operator driven over-the-air updates enable automated updates much like you are accustomed to receiving for your smart phone. What follows is a a technical exploration of the OpenShift over the air updates implementation.

# Operators All the Way Down

[![Computer Operator](/images/the-face-of-a-computer-operator-from-the-2134th-communications-squadron-is-2ca9c9.jpg)](https://nara.getarchive.net/media/the-face-of-a-computer-operator-from-the-2134th-communications-squadron-is-2ca9c9){: .align-center}

**What is an "Operator"?**

> [An Operator is][1] a method of packaging, deploying, and managing a _Kubernetes application_.

**What is a "Kubernetes application"?**

> [A Kubernetes application is][1] an app that is both deployed on Kubernetes and managed using the Kubernetes APIs.

By creating APIs and [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) for all aspects of a cluster, OpenShift essentially turns the cluster into a "kubernetes application".
{:style="clear center"}

For example here is a list of machine configuration related CRDs now available for management by Kubernetes.

{% highlight shell %}
$ oc api-resources --api-group=machineconfiguration.openshift.io
NAME                      SHORTNAMES   APIGROUP                            NAMESPACED   KIND
containerruntimeconfigs   ctrcfg       machineconfiguration.openshift.io   false        ContainerRuntimeConfig
controllerconfigs                      machineconfiguration.openshift.io   false        ControllerConfig
kubeletconfigs                         machineconfiguration.openshift.io   false        KubeletConfig
machineconfigpools        mcp          machineconfiguration.openshift.io   false        MachineConfigPool
machineconfigs            mc           machineconfiguration.openshift.io   false        MachineConfig
{% endhighlight %}

# Operators Manage the Workloads

The [Operator SDK][2] behind this work leverages technologies including Ansible, Helm, and Golang to bundle software and the logic to drive it into a Kubernetes Operator. OpenShift enables easy installation of community and ISV applications from customizatble catalog sources such as [OperatorHub](https://operatorhub.io/) and [Red Hat Marketplace](https://marketplace.redhat.com/).

After installation of a workload operator, the update or removal is managed by the [Operator Lifecycle Manager][11] or OLM. When a new release of the operator is published the OLM facilitates an update to the new operator bundle.

You can see the list of installed optional workload operators with the `oc get operators` command.

{% highlight shell %}
$ oc get operators
NAME                                                  AGE
advanced-cluster-management.open-cluster-management   65d
argocd-operator.dale                                  21d
container-security-operator.openshift-operators       54d
kubevirt-hyperconverged.openshift-cnv                 71d
podium-operator-bundle.dale                           54d
splunk-certified.splunk                               56d
{% endhighlight %}

This post will focus less on workload operators and more on cluster operators.

# Operators Manage the Platform

There is a [set of operators][6] that do not require manual installation. These operators comprise the services such as DNS, etcd, monitoring, and cloud API automation that make up the OpenShift cluster.

These are called Cluster Operators, and you can see them with the `oc get clusteroperators` command.

{% highlight shell %}
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.6.12    True        False         False      5m9s
cloud-credential                           4.6.12    True        False         False      102d
cluster-autoscaler                         4.6.12    True        False         False      102d
config-operator                            4.6.12    True        False         False      102d
console                                    4.6.12    True        False         False      2d9h
csi-snapshot-controller                    4.6.12    True        False         False      13h
dns                                        4.6.12    True        False         False      89d
etcd                                       4.6.12    True        False         False      102d
image-registry                             4.6.12    True        False         False      82d
ingress                                    4.6.12    True        False         False      7d18h
insights                                   4.6.12    True        False         False      102d
kube-apiserver                             4.6.12    True        False         False      102d
kube-controller-manager                    4.6.12    True        False         False      102d
kube-scheduler                             4.6.12    True        False         False      102d
kube-storage-version-migrator              4.6.12    True        False         False      74m
machine-api                                4.6.12    True        False         False      102d
machine-approver                           4.6.12    True        False         False      102d
machine-config                             4.6.12    True        False         False      4m30s
marketplace                                4.6.12    True        False         False      3d17h
monitoring                                 4.6.12    True        False         False      84m
network                                    4.6.12    True        False         False      102d
node-tuning                                4.6.12    True        False         False      6d18h
openshift-apiserver                        4.6.12    True        False         False      4m57s
openshift-controller-manager               4.6.12    True        False         False      10d
openshift-samples                          4.6.12    True        False         False      6d18h
operator-lifecycle-manager                 4.6.12    True        False         False      102d
operator-lifecycle-manager-catalog         4.6.12    True        False         False      102d
operator-lifecycle-manager-packageserver   4.6.12    True        False         False      7m22s
service-ca                                 4.6.12    True        False         False      102d
storage                                    4.6.12    True        False         False      77d
{% endhighlight %}

By examining these cluster operators with `oc describe` you can deduce a sense of their purpose and context. 
For example you can see that the "authentication" clusteroperator has a relation to the namespace "openshift-authentication". This would be a logical place to look for further details such as events and pod logs.

{% highlight shell %}
$ oc describe clusteroperator/authentication
{% endhighlight %}
{% highlight yaml %}
Name:         authentication
Namespace:
Labels:       <none>
Annotations:  exclude.release.openshift.io/internal-openshift-hosted: true
              include.release.openshift.io/self-managed-high-availability: true
API Version:  config.openshift.io/v1
Kind:         ClusterOperator
Metadata:
  snip...
Status:
  Related Objects:
    Group:      operator.openshift.io
    Name:       cluster
    Resource:   authentications
    Group:      config.openshift.io
    Name:       cluster
    Resource:   authentications
    Group:      config.openshift.io
    Name:       cluster
    Resource:   infrastructures
    Group:      config.openshift.io
    Name:       cluster
    Resource:   oauths
    Group:      route.openshift.io
    Name:       oauth-openshift
    Namespace:  openshift-authentication
    Resource:   routes
    Group:
    Name:       oauth-openshift
    Namespace:  openshift-authentication
    Resource:   services
    Group:
    Name:       openshift-authentication
    Resource:   namespaces
    Group:
    Name:       openshift-authentication-operator
    snip...
  Versions:
    Name:     oauth-apiserver
    Version:  4.6.12
    Name:     operator
    Version:  4.6.12
    Name:     oauth-openshift
    Version:  4.6.12_openshift
{% endhighlight %}

These operators watch for specific custom resources that they understand how to implement. For example, the `authentication` operator understands how to turn an `authentication` custom resource into an appropriately configured service.

{% highlight shell %}
$ oc api-resources --api-group=config.openshift.io --sort-by=name
NAME               SHORTNAMES   APIGROUP              NAMESPACED   KIND
apiservers                      config.openshift.io   false        APIServer
authentications                 config.openshift.io   false        Authentication
builds                          config.openshift.io   false        Build
clusteroperators   co           config.openshift.io   false        ClusterOperator
clusterversions                 config.openshift.io   false        ClusterVersion
consoles                        config.openshift.io   false        Console
dnses                           config.openshift.io   false        DNS
featuregates                    config.openshift.io   false        FeatureGate
images                          config.openshift.io   false        Image
infrastructures                 config.openshift.io   false        Infrastructure
ingresses                       config.openshift.io   false        Ingress
networks                        config.openshift.io   false        Network
oauths                          config.openshift.io   false        OAuth
operatorhubs                    config.openshift.io   false        OperatorHub
projects                        config.openshift.io   false        Project
proxies                         config.openshift.io   false        Proxy
schedulers                      config.openshift.io   false        Scheduler
{% endhighlight %}

To understand how to define an "authentication", in addition to the product documentation, you can refer to the `spec` in [the API documentation](https://docs.openshift.com/container-platform/4.6/rest_api/config_apis/authentication-config-openshift-io-v1.html) Or take advantage of the explain command eg. `oc explain DNS; oc explain DNS.spec`.

## Cluster Version Operator

These [ClusterOperators][12] like "authentication" and "console" report their status and are kept in sync by the [Cluster Version Operator](https://github.com/openshift/cluster-version-operator). The CVO is responsible for orchestrating over the air updates to the cluster. By acting on a payload representing the artifacts which make up an OpenShift release, the CVO ensures that everything from the RHEL CoreOS version on the nodes to the OpenShift web console are running on the same release.

So how does the CVO learn what upgrades are available?

### OpenShift Update Service 

The updates made available to OpenShift are published using a protocol dubbed [Cincinnati](https://github.com/openshift/cincinnati/). The update service takes advantage of integration tests and telemetry data to present only the updates that are as reliable as possible.

As updates graduate from _candidate_ to _fast_ to _stable_ or possibly _EUS_ they are presented for consumption in the corresponding release [channel](https://github.com/openshift/cincinnati-graph-data/tree/master/channels). If a an issue is discovered after release through testing or telemetry feedback, a release may be pulled and no longer show up in the graph as a possible update target. This is why it is important to never force an upgrade.

More detail may be found in 
[the documentation][5] and [this blog post][13] describing support for a locally hosted OpenShift Update Service.

The source of our update graph can be seen in ClusterVersion resource nameed "version". An alternative command would be `oc describe clusterversion`. Notice this cluster is tracking the _stable-4.6_ channel and retrieving the graph from `https://api.openshift.com/api/upgrades_info/v1/graph`.

{% highlight shell %}{% raw %}
$ oc get ClusterVersion/version -o jsonpath="{.spec}" | jq
{% endraw %}{% endhighlight  %}
{% highlight json %}{% raw %}
{
  "channel": "stable-4.6",
  "clusterID": "e640ac63-7a14-4f72-99c5-e363a071e2d8",
  "desiredUpdate": {
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:5c3618ab914eb66267b7c552a9b51c3018c3a8f8acf08ce1ff7ae4bfdd3a82bd",
    "version": "4.6.12"
  },
  "upstream": "https://api.openshift.com/api/upgrades_info/v1/graph"
}
{% endraw %}{% endhighlight %}

The `oc adm upgrade` command will use the information from that graph and present any suitable updates.
[Red Hat OpenShift Container Platform Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_channel) generates a dynamic graphical representation of the update options.

{% highlight shell %}
$ oc adm upgrade
Cluster version is 4.6.12

Updates:

VERSION IMAGE
4.6.13  quay.io/openshift-release-dev/ocp-release@sha256:8a9e40df2a19db4cc51dc8624d54163bef6e88b7d88cc0f577652ba25466e338
4.6.15  quay.io/openshift-release-dev/ocp-release@sha256:b70f550e3fa94af2f7d60a3437ec0275194db36f2dc49991da2336fe21e2824c
{% endhighlight %}

## Examining the Content of a Release Payload

We can get some info about a release, including the list of container images providing cluster operators, with the `oc adm release info` command.

{% highlight shell %}
$ oc adm release info \
quay.io/openshift-release-dev/ocp-release@sha256:8a9e40d
f2a19db4cc51dc8624d54163bef6e88b7d88cc0f577652ba25466e338
Name:      4.6.13
Digest:    sha256:8a9e40df2a19db4cc51dc8624d54163bef6e88b7d88cc0f577652ba25466e338
Created:   2021-01-20T13:12:02Z
OS/Arch:   linux/amd64
Manifests: 444

Pull From: quay.io/openshift-release-dev/ocp-release@sha256:8a9e40df2a19db4cc51dc8624d54163bef6e88b7d88cc0f577652ba25466e338

Release Metadata:
  Version:  4.6.13
  Upgrades: 4.5.21, 4.5.22, 4.5.23, 4.5.24, 4.5.27, 4.5.28, 4.6.1, 4.6.3, 4.6.4, 4.6.6, 4.6.8, 4.6.9, 4.6.10, 4.6.12
  Metadata:
    url: https://access.redhat.com/errata/RHBA-2021:0171

Component Versions:
  kubernetes 1.19.4
  machine-os 46.82.202101191342-0 Red Hat Enterprise Linux CoreOS

Images:
  NAME                                           DIGEST
  aws-ebs-csi-driver                             sha256:8f34b0cc5c7554bdfacee78ebcb5747c22dd1bf72eb4dd6007c35830e9062106
  aws-ebs-csi-driver-operator                    sha256:940d4757e6e0a603ef4fafd3d2772306b1d54f318696a35321e46ecaf2998284
  aws-machine-controllers                        sha256:a1d91b36f474b371120ae5af9c67ef97fb06cffe9d6c96ff0d2367c7fe239a43
  aws-pod-identity-webhook                       sha256:66bc5e8411973292c5f155ff5cb67569d2c081499dbf89f5c708504afa1ff600
  azure-machine-controllers                      sha256:f40212ae38936c094dc16dbf2f4d1fefa5f0ee6b68e78afe72af3cfc6b0ce8f6
  baremetal-installer                            sha256:8d2e5b267d9c45a2d808f6437023aceafad8c707cb94e57e325e3f0b2beadd62
  baremetal-machine-controllers                  sha256:36b69cc3989b1b486e7078b544fe7316e92d92ab0f1a139bcb7b1acdd1d54f14
  baremetal-operator                             sha256:ca65a235189a2929df9615995dd5f27a9ab3cda95a311cda9421cc21489e15be
  baremetal-runtimecfg                           sha256:34b95e9d96daeb80c08dbdf8a66f5b8d4a2e51a86879edd91d7bb0b40df67093
  cli                                            sha256:cf339d3bac5ecf4ee836f72bd1b3c4f0f7b0553dda63c3e73e6e2a7f1d1560ae
  cli-artifacts                                  sha256:32b6f0648ed05d5e0109d651d32093209de7db8a116fbe25e0250fac65bd2901
  cloud-credential-operator                      sha256:e490c84ffcfd6a07589eeb56600e04afc2e58edb3459643bd569be50e66e6061
  cluster-authentication-operator                sha256:baa7275273e6a4e2adb75aced5485c880368d9260df03f832a2b0a4c6cb194e3
  ...snip...
{% endhighlight %}

Pro Tip: `oc adm release info quay.io/openshift-release-dev/ocp-release:4.6.13-x86_64` would have also worked.

## Managing the Nodes

The machines which make up the cluster are also managed by operators. The `machine-api` operator interfaces with the underlying infrastructure provider (OpenStack, AWS, etc) to provision machines on Day 1. The `machine-config` operator manages the resources that provide Day 1 and Day 2 configuration.

When a machine is booted it is provided a URL to the machine config server which serves up an [Ignition Config][4]. Ignition can be thought of as a combination of Kickstart and Cloud-Init. The config will include a reference to the RHCOS image to install on the host and every file and configuration detail to be applied.  Machines are grouped into pools and subscribe to corresponding MachineConfigPools like _master_ or _worker_ in this example.

![Machine Config Server](/images/openshift-mcs-slide-640.png)

Since Ignition runs in the initial RAM disk (initrd), it has the ability to make low level changes to storage and networking without requiring a reboot.
If there are any errors reported by Ignition during the provisioning phase, the machine will not be placed into service.

The `machine-config` ClusterOperator has as an operand the `machine-config-daemon`. [The MCD][8] is deployed as a daemonset to all nodes and prevents configuration drift by checking in with the `machine-config-server`. When a change is detected between the currently applied machine configuration and the desired configuration the `machine-config-controller`. [The MCC][19] will coordinate the application of the change in a controlled manner.

OpenShift 4 considers nodes to be immutable. That is to say, if they break or require a change they should be "replaced" or configured in whole rather than in part. A practical effect of this is that the only way to affect a change is to create a [MachineConfig][15]  any change to a machine configuration requires a reapplication of all changes by way of a node reboot. The number of nodes impacted simultaneously may be configured with the `maxUnavailable` attribute of a MachineConfigPool which will be honored by the MCC.

### Updating Node Operating Systems with libostree

As shown above, the OpenShift release image contains a reference to a [specific release of RHCOS][17] eg. `machine-os 46.82.202101191342-0 Red Hat Enterprise Linux CoreOS`. 

RHEL CoreOS like RHEL Atomic Host uses [libostree][18] to apply operating system updates in a transactional manner. During an update the MCD downloads the container image provided in the `osImageURL` attribute of the Ignition config. This is image essentially a wrapper around an OSTree commit which is extracted and written to 
`/ostree/deploy/rhcos/deploy/<hash>`. The boot configuration is then updated to point to the new OSTree.

{%highlight shell %}
sh-4.4# rpm-ostree status
State: idle
AutomaticUpdates: disabled
Deployments:
* pivot://quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:6069ffcc5dcd227c867a86970f3d2076221081a81b6c92fbecfd044ef160161e
              CustomOrigin: Managed by machine-config-operator
                   Version: 46.82.202101131942-0 (2021-01-13T19:45:25Z)
                    Commit: 046de410ba240083996855d797207e09374bbfdf65e5be34baa1e1f1ab49918a
                            |- art-rhaos-4.6 (2021-01-13T17:25:39Z)
                            |- rhel8-fast-datapath (2020-11-23T19:03:06Z)
                            |- rhel8-baseos (2021-01-12T14:14:01Z)
                            `- rhel8-appstream (2021-01-13T10:43:31Z)
                    Staged: no
                 StateRoot: rhcos

  pivot://quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:4c5b83a192734ad6aa33da798f51b4b7ebe0f633ed63d53867a0c3fb73993024
              CustomOrigin: Managed by machine-config-operator
                   Version: 46.82.202012051820-0 (2020-12-05T18:24:10Z)
                    Commit: 7e014b64b96536487eb3701fa4f0e604697730bde2f2f9034c4f56c024c67119
                            |- art-rhaos-4.6 (2020-12-05T14:07:54Z)
                            |- rhel8-fast-datapath (2020-11-23T19:03:06Z)
                            |- rhel8-baseos (2020-11-23T17:53:53Z)
                            `- rhel8-appstream (2020-11-30T10:18:29Z)
                 StateRoot: rhcos
{% endhighlight %}

#### MCD Troubleshooting

The MCD is of course running in a pod, so you can spy on its logs on each node and observe its actions or check for problems. For example, while unusual (I've seen the following once), if you found a problem with the boot config like this...

{%highlight shell %}
oc project openshift-machine-config-operator
for POD in `oc get pod -o name -l k8s-app=machine-config-daemon`; do
  echo ----
  echo Checking $POD for osImageURL mismatch
  oc get $POD -o wide
  echo
  oc logs $POD -c  machine-config-daemon  | grep "expected target osImageURL" | tail -1
done
...snip...
----
Checking pod/machine-config-daemon-nn5p8 for osImageURL mismatch
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE                  NOMINATED NODE   READINESS GATES
machine-config-daemon-nn5p8   2/2     Running   0          15m   192.168.4.63   vipi-7zc6h-master-1   <none>           <none>

E0209 18:23:14.965590  116436 writer.go:135] Marking Degraded due to: unexpected on-disk state validating against \
 rendered-master-ff38f849e21009792e7db36fe2c27ef9: expected target osImageURL "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:6069ffcc5dcd227c867a86970f3d2076221081a81b6c92fbecfd044ef160161e",\
 have "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:4c5b83a192734ad6aa33da798f51b4b7ebe0f633ed63d53867a0c3fb73993024"
{% endhighlight %}

You can manually update the boot configuration.

{%highlight shell %}
oc debug node/vipi-7zc6h-master-1
sh-4.4# chroot /host
sh-4.4# rpm-ostree status -v
...snip...
sh-4.4# /run/bin/machine-config-daemon pivot quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:6069ffcc5dcd227c867a86970f3d207622108
1a81b6c92fbecfd044ef160161e
sh-4.4# reboot
{% endhighlight %}


Finally when the node is rebooted with the node annotations updated by the MachineConfigControllerare updated to reflect the state.
When the values of `machineconfiguration.openshift.io/currentConfig` and `machineconfiguration.openshift.io/desiredConfig` match the MCD is happy and so are we!

# Summary

This was admittedly heavy on external links and possibly "hand wavy" at points, but I hope it provides you a better sense of just exactly how OpenShift Over The Air Updates automate the administration of your entire Kubernetes cluster.

## Watch a Video Demonstration

For more information on this topic including a video demonstration of the OpenShift 4 upgrade process, please see [my BrightTALK presentation][3] from September, 2020.

[![OpenShift Over-The-Air Updates BrightTALK](/images/openshift-ota-brighttalk.png)][3]


## References

* [Red Hat OpenShift Cluster Services: Over-The-Air Updates Webinar][3]
* [What are Operators?][1]
* [What are Cluster Operators?][12]
* [Cluster Operators Reference][6]
* [Operator SDK][2]
* [Ignition Configuration Specifications][4]
* [OpenShift Upgrading Between Minor Releases][5]
* [Machine Configuration Daemon][8]
* [Machine Config Operator and MachineConfigs][15]
* [Machine Config Operator and MachineConfigController][19]
* [Machine Config Operator and OS Updates][16] includes some bootstrap tidbits
* [OpenShift continuous delivery pipeline artifacts][14]
* [CoreOS Assembler builds CoreOS Images][17]
* [OSTree or "libostree"][18]


[1]: https://docs.openshift.com/container-platform/latest/operators/olm-what-operators-are.html "What are Operators? OpenShift Documentation"
[2]: https://github.com/operator-framework/operator-sdk "Operator SDK Repository"
[3]: https://www.brighttalk.com/webcast/14777/433967 "Over The Air Updates BrightTALK Video Presentation"
[4]: https://coreos.github.io/ignition/specs/ "Ignition Specification"
[5]: https://docs.openshift.com/container-platform/latest/updating/updating-cluster-between-minor.html "Upgrading Between Minor Releases OpenShift Documentation"
[6]: https://docs.openshift.com/container-platform/latest/operators/operator-reference.html "Red Hat Operators"
[8]: https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfigDaemon.md "MachineConfigDaemon"
[9]: https://coreos.com/blog/introducing-operator-framework "Introducing Operator Framework"
[11]: https://github.com/operator-framework/operator-lifecycle-manager "Operator Lifecycle Manager"
[7]: https://github.com/operator-framework/ "Operator Framwork Github Organization"
[10]: https://www.redhat.com/en/blog/openshift-container-platform-4-how-does-machine-config-pool-work
[12]: https://github.com/openshift/cluster-version-operator/blob/master/docs/dev/clusteroperator.md "Cluster Operator"
[13]: https://www.openshift.com/blog/openshift-update-service-update-manager-for-your-cluster "openshift-update-service-update-manager-for-your-cluster"
[14]: https://openshift-release.apps.ci.l2s4.p1.openshiftapps.com/ "OpenShift Continuous Delivery Pipeline Artifacts"
[15]: https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfiguration.md "MachineConfig"
[16]: https://github.com/openshift/machine-config-operator/blob/master/docs/OSUpgrades.md "OS Updates"
[17]: https://github.com/coreos/coreos-assembler "CoreOS Assembler"
[18]: https://github.com/ostreedev/ostree "OSTree"
[19]: https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfigController.md "MachienConfigController"
