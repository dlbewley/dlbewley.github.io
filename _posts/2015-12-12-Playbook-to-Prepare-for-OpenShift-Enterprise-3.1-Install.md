---
title: Ansible Playbook to Prepare for OpenShift Enterprise 3.1
layout: post
tags:
 - ansible
 - openshift
---

This playbook is written for RHEL 7.2 and OSE v3.1. It will perform the following steps which should take place before running the [openshift-ansible byo playbook](https://github.com/openshift/openshift-ansible/blob/master/playbooks/byo/config.yml).

- Install prerequisite RPMs like docker, python, etc.
- Persist the systemd journal for easier debugging
- Setup docker ephemeral storage on 2nd disk
- [Turn off swap](https://docs.openshift.com/enterprise/3.1/admin_guide/overcommit.html#disable-swap-memory)
- Enable use of NFS in selinux

# Prerequisites #

See my [Testing OpenShift Enterprise V3](http://guifreelife.com/blog/2015/07/28/Testing-OpenShift-Enterprise-V3) post for the prereqs.

# The Playbook #

The lastest version is [available here](https://github.com/dlbewley/playbook-openshift/blob/master/prep.yml).

```yaml
---
# file: prep.yml
# @dlbewley
# Playbook to prep OpenShift Enterprise hosts for installation. Run this before
# the openshift-ansible byo playbook.

- hosts: all
  #-----

  vars:
  #-----

  - docker_storage_setup: |
      DEVS=/dev/vdb
      VG=docker-vg

  tasks:
  #-----

  - name: Remove NetworkManager
    yum:
      name: NetworkManager
      state: absent

  - name: Install prereqs
    yum:
      name: "{{ item }}"
      state: present
    with_items:
      - atomic-openshift-utils
      - bind-utils
      - bridge-utils
      - docker
      - gcc
      - git
      - iptables-services
      - net-tools
      - openssh-clients
      - python
      - python-virtualenv
      - wget

  - name: Install life enhancers on master
    yum:
      name: "{{ item }}"
      state: present
    with_items:
      - bash-completion
      - deltarpm
      - etcd
      - tmux
      - vim-enhanced
    when: "'masters' in group_names"

  - name: Make journald logs persistent to assist troubleshooting
    file:
      name: /var/log/journal
      state: directory
    notify: restart journald

  # openshift-ansible/roles/openshift_node/tasks/storage_plugins/nfs.yml
  # enables sebool 'virt_use_nfs', but i still had to use 'all_squash' rather
  # than 'root_squash' on the NFS export. 
  # Will the following fix that? Not yet tested.
  - name: OpenShift use nfs files
    seboolean:
      name: openshift_use_nfs
      state: yes
      persistent: yes
    when: "'nodes' in group_names"

  # https://docs.openshift.com/enterprise/3.1/admin_guide/overcommit.html#disable-swap-memory
  - name: Remove swaps from fstab
    lineinfile:
      dest: /etc/fstab
      regexp: '^/[\S]+\s+swap\s+swap'
      state: absent
    notify: disable swap
    when: "'nodes' in group_names"

  - name: Setup docker storage config
    copy:
      content: "{{ docker_storage_setup }}"
      dest: /etc/sysconfig/docker-storage-setup
    notify: docker storage setup
    when: "'nodes' in group_names"


  handlers:
  #-------

  - name: disable swap
    command: swapoff --all
    ignore_errors: yes

  - name: restart journald
    service:
      name: systemd-journald
      state: restarted
  
  - name: docker storage setup
    command: docker-storage-setup
```
