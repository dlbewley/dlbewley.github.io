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

The OpenShift Container Platform (OCP) can run on many types of infrastructure; from a Docker contrainer, to a single VM, to a fleet of baremetal or VMs on an infrastructure provider such as RHV, VMware, Amazon EC2, Google Compute Engine, or OpenStack Platform (OSP). This post is to document my experimentation with setting up OCP on OSP.

# Doc Overview #

So where are the docs?

The Reference Architecture [2017 - Deploying Red Hat OpenShift Container Platform 3.4 on Red Hat OpenStack Platform 10](https://access.redhat.com/articles/3133011) derives from the Redhat [OpenShift on OpenStack Github repo](https://github.com/redhat-openstack/openshift-on-openstack)  provides the orchestration templates to stand up a infrastructure stack to run OpenShift on. At the moment the last releases are OCP 3.6 and OSP 11. Let's target that even if it "isn't supported".

Antonio Gallego has created a script to [prepare OpenStack for OpenShift](https://github.com/antoniogallegosaez/openshift_on_openstack_preparer). Does this overlap with the openshift-on-openstack repo though? The openshift-ansible repo includes a [openstack dynamic inventory](https://github.com/openshift/openshift-ansible/blob/master/inventory/openstack/hosts/openstack.py) module, but it is not yet clear to me how best to utilize it. For my production cluster, I tend to keep my intentory synchronized to [hosts.ose.example](https://github.com/openshift/openshift-ansible/blob/master/inventory/byo/hosts.ose.example).

The openshift-ansible-contrib project has a [openstack-stack](https://github.com/openshift/openshift-ansible-contrib/tree/master/roles/openstack-stack) role for deploying OpenShift to OpenStack, I'm not sure of the state, but there is a [reference-architecture/osp-dns](https://github.com/openshift/openshift-ansible-contrib/tree/master/reference-architecture/osp-dns) to deploy a DNS infra suitable for testing.

While heat and the ansible playbook will do this for you, it is interesting to look at the [CLI commands](https://github.com/openshift/openshift-ansible-contrib/tree/master/reference-architecture/osp-cli) to configure OpenStack for OpenShift. This scripts are sprinkled through the ref arch document as well.

OpenShift its self needs to be [configured for OpenStack](https://docs.openshift.com/container-platform/latest/install_config/configuring_openstack.html) to make use of storage and other services provided by OpenStack. The [OpenShift Ansible playbook](https://github.com/openshift/openshift-ansible) is used to install and configure OpenShift on any platform including OpenStack and the settings will be placed in the playbook host inventory file.

# Networking Overview #

OpenShift highly available routing is somewhat complex on its own.

[![OpenShift HA Routing](/images/thumb/openshift-ha-cluster-routing.png)](http://guifreelife.com/blog/2016/03/01/OpenShift-3-HA-Routing)

Toss in the even more complex OpenStack networking, and well, hopefully you are not starting from scratch.

[![OpenStack Network Diagram](/images/thumb/openstack-network-pub.png)](http://guifreelife.com/blog/2017/08/14/OpenStack-Network-Diagram)

The reference architecture forgoes the [OpenShift SDN with ovs-subnet plugin](https://docs.openshift.com/container-platform/latest/architecture/additional_concepts/sdn.html) and [uses Flannel](https://docs.openshift.com/container-platform/latest/architecture/additional_concepts/flannel.html). Notes on [configuring Flannel networking](https://docs.openshift.com/container-platform/latest/install_config/configuring_sdn.html#using-flannel).

[![Openshift on OpenStack Reference Arch Diagram](https://raw.githubusercontent.com/redhat-openstack/openshift-on-openstack/master/graphics/architecture.png)](https://github.com/redhat-openstack/openshift-on-openstack/blob/master/graphics/architecture.png)

There are some drawbacks to using Flannel when it comes to isolation. An interesting alternative to Flannel could be [Project Calico](https://www.projectcalico.org/) which uses BGP routing amongst containers while also supporting microsegmentation of traffic. [Tigera.io](https://www.tigera.io) develops and supports Calico commercially.

## Networking Details ##

There will be 3 OpenStack networks in use:

- A _public_ network for external access to the OpenStack / OpenShift services eg: `N5: External` or `N8: Provider`
  To do a HA setup, I think I need a
- A _control_ network for OpenShift node communication eg:
- A _tenant_ network for container communicatios

## Instance Details ##

Host                  | IP                        | Description
----------------------|---------------------------|------------
openshift.ocp3.example.com | 10.19.x.y (Load Balancer) | Web console and API endpoint
*.ocp3.example.com    | 10.19.x.y (Router)        | OpenShift routes to services handled by haproxy.
bastion               |                           | Operator access and Ansible management point
master-01             |                           | One of three redundant OpenShift Masters
master-02             |                           | One of three redundant OpenShift Masters
master-03             |                           | One of three redundant OpenShift Masters
node-01               |                           | One of two redundant Infrastructure nodes
node-02               |                           | One of two redundant Infrastructure nodes
node-03               |                           | One of two redundant Application nodes
node-04               |                           | One of two redundant Application nodes
etcd?                 |                           | Etcd is not called out in the ref arch. Does it assume all-in-one master?


- Public network for external admin access
- Control network


# Infrastructure Setup #

The following steps will need to take place whether by hand, or with the benefit of the [OpenStack Heat](https://www.openstack.org/software/releases/ocata/components/heat) templates from the [OpenShift on OpenStack repo](https://github.com/redhat-openstack/openshift-on-openstack).

- [x] From the OpenStack Director host, RHEL image in glance from `rhel-guest-image-7` rpm. (Convert to RAW is using Ceph RBD CoW)

```bash
$ sudo yum -y install rhel-guest-image-7 # or download newer from https://access.redhat.com
$ cp -p /usr/share/rhel-guest-image-7/rhel-guest-image-7*.qcow2 /tmp/
$ virt-customize \
    -a /tmp/rhel-guest-image-7*.qcow2 \
    --root-password password:<default_root_password>
$ qemu-img convert \
    -f qcow2 \
    -O raw \
    /tmp/rhel-guest-image-7*.qcow2 \
    /tmp/rhel7.raw
$ openstack image create \
    rhel7 \
    --container-format bare \
    --disk-format raw \
    --file /tmp/rhel7.raw \
    --public
```

- deploy 8 virtual machines on network

- provision DNS dynamic domain See [appendix B](https://access.redhat.com/documentation/en-us/reference_architectures/2017/html-single/deploying_red_hat_openshift_container_platform_3.4_on_red_hat_openstack_platform_10/#appendix-b) and [openshift-ansible-contrib/roles/dns-records](https://github.com/openshift/openshift-ansible-contrib/blob/master/roles/dns-records/tasks/main.yml)

- configure [OpenShift Ansible](https://github.com/openshift/openshift-ansible) inventory file
- deploy OpenShift

## OpenStack Prerequisites ##

Many of these steps are included in [openshift-ansible-contrib/reference-architecture/osp-cli](https://github.com/openshift/openshift-ansible-contrib/tree/master/reference-architecture/osp-cli)

- [x] Increase keystone token expiration to 7200s to ensure OpenShift Ansible can complete. See [appendix](https://access.redhat.com/documentation/en-us/reference_architectures/2017/html-single/deploying_red_hat_openshift_container_platform_3.4_on_red_hat_openstack_platform_10/#appendix-c) and [solution 1527533](https://access.redhat.com/solutions/1527533)

```yaml
#  ansible playbook to extend keystone token expiration
- hosts: controller
  become: true
  become_user: root

  tasks:
  - name: who am i
    debug:
      msg: "{{ansible_fqdn}}"

  - name: configure keystone token expiration
    lineinfile:
      dest: /etc/keystone/keystone.conf
      regexp: '^expiration = .*'
      line: 'expiration = 7200'
      backup: yes
    notify: restart keystone

  handlers:
  - name: restart keystone
    service:
      name: httpd
      state: restarted
```

- [x] OpenStack tenant to run in: eg. `ocp3`

```bash
$ openstack project create ocp3 --enable
$ openstack user create ocp3 --email dlbewley@example.com --project ocp3 --enable --password <password>
# create ocp3rc with credentials
```

- [x] [Increase Quota](https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/quota.html) to meet [recommendations](https://access.redhat.com/documentation/en-us/reference_architectures/2017/html/deploying_red_hat_openshift_container_platform_3.4_on_red_hat_openstack_platform_10/deploying_using_heat#project_quotas)

```bash
$ openstack quota show ocp3
...
$ openstack quota set ocp3 \
              --cores 60 \
              --gigabytes 2000 \
              --instances 20 \
              --ram $(( 450 * 1024 )) \
              --volumes 30
```

- [x] An SSH keypair pre-loaded in nova

```bash
# save the private key for later
$ openstack keypair create ocp3 > ~/.ssh/ocp3.key
$ openstack keypair show ocp3 --public-key > ~/.ssh/ocp3.pub
```

- [ ] Openstack user account in tenant: eg.`ocp_ops`
 - Grant `member` role to install OpenShift manually
 - Grant `heat_stack_owner` role to install using Heat

- A Red Hat Enterprise Linux update subscription credentials (user/password or Satellite server)

- A publicly available neutron network for inbound access to router (infra nodes), masters, bastion,
A pool of floating IP addresses to provide inbound access points for instances
A host running haproxy to provide load-balancing
A host running bind with a properly delegated sub-domain for publishing, and a key for dynamic updates.
An existing LDAP or Active Directory service for user identification and authentication

## Deploy a Dyanmic DNS Server ##

During testing [create a DNS server](https://access.redhat.com/documentation/en-us/reference_architectures/2017/html/deploying_red_hat_openshift_container_platform_3.4_on_red_hat_openstack_platform_10/appendix-b) in the project which we can be updated using `nsupdate`.

```bash
git clone https://github.com/openshift/openshift-ansible-contrib
```

- [ ] Create a `dns-vars.yaml` for feeding to [deploy-dns.yaml](https://github.com/openshift/openshift-ansible-contrib/blob/master/reference-architecture/osp-dns/deploy-dns.yaml) playbook

```yaml
---
domain_name: ocp3.example.com
contact: admin@ocp3.example.com
# real DNS servers from environment
dns_forwarders: [10.x.x.41, 10.x.x.2]
update_key: "NOT A REAL KEY"
slave_count: 2

stack_name: dns-service
external_network: public

image: rhel7
flavor: m1.small
ssh_user: cloud-user
ssh_key_name: ocp3

# NOTE: For Red Hat Enterprise Linux:
# rhn_username: "rhnusername"
# rhn_password: "NOT A REAL PASSWORD"
# rhn_pool: "pool id string"
# Either RHN or Sat6
sat6_hostname: ""
sat6_organization: ""
sat6_activationkey: ""
```

- [ ] Run `deploy-dns.sh` like this

```bash
#!/bin/bash
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook --private-key ocp3.key -e @dns-vars.yaml -vv\
        openshift-ansible-contrib/reference-architecture/osp-dns/deploy-dns.yaml | tee dns-deploy.log
```

My network deployed, but I hit an error deploying the instances so I [opened an issue](https://github.com/openshift/openshift-ansible-contrib/issues/699)

```
17-08-27 02:21:14Z [dns-service.hosts]: CREATE_FAILED  ResourceInError: resources.hosts.resources.slaves.resources.slaves.resources[1].resources.host: Went to status ERROR due to \"Message: ServerGroup policy is not supported: ServerGroupAntiAffinityFilter not configured, Code: 400\"", "2017-08-27 02:21:14Z [dns-service]: CREATE_FAILED  Resource CREATE failed: ResourceInError: resources.hosts.resources.slaves.resources.slaves.resources[1].resources.host: Went to status ERROR due to \"Message: ServerGroup policy is not supported: ServerGroupAntiAffinityFilter not configured, Code: 400\""]}
```

# Heat Deployment #

[Deploy OpenShift on OpenStack using Heat](https://access.redhat.com/documentation/en-us/reference_architectures/2017/html-single/deploying_red_hat_openshift_container_platform_3.4_on_red_hat_openstack_platform_10/#deploying_using_heat)

- Make the heat templates available. If the `openshift-heat-templates` RPM is not reachable, clone the repo and

```bash
sudo yum -y install python-heatclient openshift-heat-templates # missing in OSP 11
# or
cd ~stack/templates
git clone https://github.com/redhat-openstack/openshift-on-openstack
```

- Get or create a `openshift_parameters.yaml` heat template. Use [this one](https://github.com/redhat-openstack/openshift-on-openstack/blob/master/openshift.yaml) or copy from example in README.adoc or copy [from the ref arch]( https://access.redhat.com/documentation/en-us/reference_architectures/2017/html/deploying_red_hat_openshift_container_platform_3.4_on_red_hat_openstack_platform_10/deploying_using_heat#heat_stack_configuration)
