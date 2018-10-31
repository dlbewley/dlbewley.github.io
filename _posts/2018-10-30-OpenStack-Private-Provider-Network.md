---
title: Creating OpenStack Provider Network for Use by a Single Project
layout: post
tags:
 - ansible
 - openstack
 - networking
---

OpenStack supports ["provider" networks](https://docs.openstack.org/install-guide/launch-instance-networks-provider.html), which are networks that pre-exist in your physical infrastructure and are "provided" to the cloud users rather than created by the user. Only an admin is permitted to create a provider network.

A prequisite is the provider network must be plumbed to the external bridge on your controller and nova nodes.

Here is an Ansible playbook to create a project, place a unshared provider network and subnet in that project. Afterwards we will grant access to the members of this project using the openstack client. It [does not appear](https://docs.ansible.com/ansible/latest/modules/list_of_cloud_modules.html#openstack) that Ansible has a OpenStack network RBAC module at this time.



{% highlight yaml %}{% raw %}
---
# file: project-tools-build.yml
# playbook to create tools-build project and network
#
# Run me as `admin` on overcloud.
# That means you may want to run like this:
#   `ansible-playbook -e cloud=admin project-tools-build.yml`

- hosts: localhost
  connection: local
  vars:
    # use python from virtualenv path
    ansible_python_interpreter: python
    cloud: admin
    admin_cloud: admin
    cloud_domain: Default
    
    projects:
      - project: tools-build
        users:
          - username: tools-build-user

    networks:
      tools-build:
        project: tools-build
        network_name: tools-build
        subnet_name: tools-build-subnet
        external: True
        shared: False
        provider_physical_network: datacentre
        provider_network_type: vlan
        provider_segmentation_id: 2
        cidr: 192.0.2.0/24
        gateway_ip: 192.0.2.254
        allocation_pool_start: 192.0.2.20
        allocation_pool_end: 192.0.2.250
        dns_nameservers:
          - 192.0.1.1
          - 192.0.1.2

  tasks:
    - name: Create projects
      os_project:
        cloud: "{{cloud}}"
        name: "{{item.project}}"
        domain_id: "{{ cloud_domain }}"
        description: "{{item.project}} Project"
        enabled: True
      with_items: "{{ projects }}"

    - name: Create users
      os_user:
        cloud: "{{ cloud }}"
        name: "{{ item.1.username }}"
        password: "{{ item.1.password | default('secret') }}"
        email: "{{ item.1.username }}@example.com"
        domain: "{{ cloud_domain }}"
        default_project: "{{ item.0.project }}"
        enabled: True
      with_subelements:
        - "{{ projects }}"
        - users

    - name: Add users as members of projects
      os_user_role:
        cloud: "{{ cloud }}"
        user: "{{ item.1.username }}"
        role: _member_
        project: "{{ item.0.project }}"
      with_subelements:
        - "{{ projects }}"
        - users

    - name: Add admin user to projects
      os_user_role:
        cloud: "{{ cloud }}"
        user: "admin"
        role: _member_
        project: "{{ item.project }}"
      with_items:
        - "{{ projects }}"

    - name: Create networks
      os_network:
        cloud: "{{admin_cloud}}"
        name: "{{item.value.network_name}}"
        project: "{{ item.value.project | default(omit) }}"
        external: "{{ item.value.external | default(True) }}"
        provider_network_type: "{{ item.value.provider_network_type | default(omit) }}"
        provider_segmentation_id: "{{ item.value.provider_segmentation_id | default(omit) }}"
        provider_physical_network: "{{ item.value.provider_physical_network | default(omit) }}"
        shared: "{{ item.value.shared | default(False) }}"
        state: present
      with_dict: "{{ networks }}"

    - name: Create subnets
      os_subnet:
        cloud: "{{admin_cloud}}"
        state: present
        name: "{{ item.value.subnet_name }}"
        network_name: "{{ item.value.network_name }}"
        project: "{{ item.value.project | default(omit) }}"
        cidr: "{{ item.value.cidr }}"
        gateway_ip: "{{ item.value.gateway_ip | default(omit) }}"
        allocation_pool_start: "{{ item.value.allocation_pool_start | default(omit) }}"
        allocation_pool_end:  "{{ item.value.allocation_pool_end | default(omit) }}"
        dns_nameservers: "{{ item.value.dns_nameservers | default(dns_nameservers) | join(',') }}"
      with_dict: "{{ networks }}"
{% endraw %}{% endhighlight %}

At this point you will have a provider network called `tools-build` located in the tools-build project, but members of that tenant will not have rights to access it, because we set `shared: False`.

[Neutron RBAC](https://docs.openstack.org/ocata/networking-guide/config-rbac.html) will enable us to selectively exclusively share this network with the `tools-build` tenant.

Perform the following as admin on the overcloud:

```bash
# get the ID for tools-build project
$ openstack project show tools-build -f value -c id
05d445c63e104f5bafafe21a8bc2a28a

# get the ID for tools-build network
$ openstack network show tools-build -f value -c id
3474ccc2-4c77-4c65-a6fb-87364ec7f135

# create an RBAC rule allowing access
$ openstack network rbac create --target-project 05d445c63e104f5bafafe21a8bc2a28a \
  --action access_as_shared \
  --type network 3474ccc2-4c77-4c65-a6fb-87364ec7f135
```

Keep in mind the network will now say `shared` is _True_ from the tools-build project, but _False_ from other projects. Examine the RBAC rules on the network to see how.

```bash
# find RBAC rules for tools-build network
$ openstack network rbac list | grep $(openstack network show tools-build -f value -c id)
| 206121dc-e7e7-4d95-a03a-40d171fce0b8 | network     | 3474ccc2-4c77-4c65-a6fb-87364ec7f135 |
| 6a73157a-d532-43b0-aabf-8cf2f2f40e97 | network     | 3474ccc2-4c77-4c65-a6fb-87364ec7f135 |

# show first rule
$ openstack network rbac show 206121dc-e7e7-4d95-a03a-40d171fce0b8
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| action            | access_as_shared                     |
| id                | 206121dc-e7e7-4d95-a03a-40d171fce0b8 |
| name              | None                                 |
| object_id         | 3474ccc2-4c77-4c65-a6fb-87364ec7f135 |
| object_type       | network                              |
| project_id        | f806371dd95d4bcf982fe801852fd996     |
| target_project_id | 05d445c63e104f5bafafe21a8bc2a28a     |
+-------------------+--------------------------------------+

# show second rule
$ openstack network rbac show 6a73157a-d532-43b0-aabf-8cf2f2f40e97
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| action            | access_as_external                   |
| id                | 6a73157a-d532-43b0-aabf-8cf2f2f40e97 |
| name              | None                                 |
| object_id         | 3474ccc2-4c77-4c65-a6fb-87364ec7f135 |
| object_type       | network                              |
| project_id        | 05d445c63e104f5bafafe21a8bc2a28a     |
| target_project_id | *                                    |
+-------------------+--------------------------------------+
```

At this point you should be able to create an instance attached to this network.

```bash
export OS_CLOUD="tools-build"
openstack server create \
  --flavor m1.small \
  --image rhel-server-7.5-x86_64-kvm.raw \
  --key-name $USER \
  --nic net-id=$(openstack network show tools-build -c id -f value) \
  tools-build-instance
```

**See Also**

- SuperUser [Tenant networks vs. provider networks in the private cloud context](http://superuser.openstack.org/articles/tenant-networks-vs-provider-networks-in-the-private-cloud-context/)
