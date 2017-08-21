---
title: Installing OpenShift on OpenStack
layout: post
tags:
 - kubernetes
 - openshift
 - openstack
 - draft
---

**This is a work in progress**

The OpenShift Container Platform (OCP) can run on many types of infrastructure. From a Docker contrainer, to a single VM, to a fleet of baremetal or VMs on an infrastructure provider such as RHV, VMware, Amazon EC2, Google Compute, or OpenStack Platform (OSP). This post is to document my experimentation with setting up OCP on OSP.

# Doc Overview #

So where are the docs?

The Reference Architecture [2017 - Deploying Red Hat OpenShift Container Platform 3.4 on Red Hat OpenStack Platform 10](https://access.redhat.com/articles/3133011) derives from the Redhat [OpenShift on OpenStack Github repo](https://github.com/redhat-openstack/openshift-on-openstack)  provides the orchestration templates to stand up a infrastructure stack to run OpenShift on.

Antonio Gallego has created a script to [prepare OpenStack for OpenShift](https://github.com/antoniogallegosaez/openshift_on_openstack_preparer). Does this overlap with the openshift-on-openstack repo though?

OpenShift its self needs to be [configured for OpenStack](https://docs.openshift.com/container-platform/latest/install_config/configuring_openstack.html) to make use of storage and other services provided by OpenStack. The [OpenShift Ansible playbook](https://github.com/openshift/openshift-ansible) is used to install and configure OpenShift on any platform including OpenStack and the settings will be placed in the playbook host inventory file.

# Networking Overview #

OpenShift highly available routing is somewhat complex on its own.
[![OpenShift HA Routing](/images/thumb/openshift-ha-cluster-routing.png)](http://guifreelife.com/blog/2016/03/01/OpenShift-3-HA-Routing)

Toss in the even more complete OpenStack networking, and well, hopefully you are not starting from scratch.
[![OpenStack Network Diagram](/images/thumb/openstack-network-pub.png)](http://guifreelife.com/blog/2017/08/14/OpenStack-Network-Diagram)

The reference architecture forgoes the [OpenShift SDN with ovs-subnet plugin](https://docs.openshift.com/container-platform/latest/architecture/additional_concepts/sdn.html) and [uses Flannel](https://docs.openshift.com/container-platform/latest/architecture/additional_concepts/flannel.html). Notes on [configuring Flannel networking](https://docs.openshift.com/container-platform/latest/install_config/configuring_sdn.html#using-flannel). 

There are some drawbacks to using Flannel when it comes to isolation. An interesting alternative to Flannel could be [Project Calico](https://www.projectcalico.org/) which uses BGP routing amongst containers.

