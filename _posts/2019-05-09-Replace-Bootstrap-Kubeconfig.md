---
title: Playbook to replace bootstrap.kubeconfig and node certificates on OpenShift 3.10 3.11
layout: post
tags:
 - openshift
 - OCP3.10
 - OCP3.11
---

If you are a serial upgrader like me, you may have found that at one point during your 3.10.xx patching (say 3.10.119) you hit this error during the data plane upgrade:

{% highlight plain %}{% raw %}
TASK [openshift_node : Approve the node] ************************************************************
task path: /usr/share/ansible/openshift-ansible/roles/openshift_node/tasks/upgrade/restart.yml:49
Using module file /usr/share/ansible/openshift-ansible/roles/lib_openshift/library/oc_csr_approve.py
...
FAILED - RETRYING: Approve the node (30 retries left).Result was: {
    "all_subjects_found": [],
    "attempts": 1,
    "changed": false,
    "client_approve_results": [],
    "client_csrs": {},
    "failed": true,
    "invocation": {
        "module_args": {
            "node_list": [
                "ose-test-node-01.example.com"
            ],
            "oc_bin": "oc",
            "oc_conf": "/etc/origin/master/admin.kubeconfig"
        }
    },
    "msg": "Could not find csr for nodes: ose-test-node-01.example.com",
...
{% endhighlight %}{% raw %}

Turns out this was because the start up of `atomic-openshift-node` failed to generate a CSR.

{% endhighlight %}{% raw %}
[root@ose-test-node-01 ~]# journalctl -xe
...
Apr 05 11:17:13 ose-test-node-01.example.com atomic-openshift-node[62930]: I0405 11:17:13.675814   62930 bootstrap.go:56] Using bootstrap kubeconfig to generate TLS client cert, key and kubeconfig file
Apr 05 11:17:13 ose-test-node-01.example.com atomic-openshift-node[62930]: I0405 11:17:13.677298   62930 bootstrap.go:86] No valid private key and/or certificate found, reusing existing private key or creating
Apr 05 11:17:13 ose-test-node-01.example.com atomic-openshift-node[62930]: F0405 11:17:13.705228   62930 server.go:235] failed to run Kubelet: cannot create certificate signing request: Unauthorized
{% endhighlight %}{% raw %}

And why is that? It is because `/etc/origin/node/bootstrap.kubeconfig` is out of date. Why is this? A bug in the upgrade apparently.

The fix is the same as it would be to replace the node certificates. There is a [handy knowledgebase article](https://access.redhat.com/support/cases/#/case/02355566) for this. 

Here is a playbook to do that.



{% highlight yaml %}{% raw %}
---
# Manually replace 3.10 or 3.11 node certificates https://access.redhat.com/solutions/3821042
#
# Run from first master. 

- hosts: primary-nodes,infra-nodes
  serial: 1
  vars:
    # only used for logging path
    target_ver: 3.10.119
    node_backup_dir: "/root/playbook-openshift/docs/upgrades/{{ target_ver }}/bootstrap-fix-{{ansible_date_time.date}}-{{ansible_date_time.hour}}/"
    new_bootstrap: "{{ node_backup_dir }}/bootstrap.kubeconfig"
    files_to_replace:
        - /etc/origin/node/bootstrap.kubeconfig
        - /etc/origin/node/certificates/kubelet-client-current.pem
        - /etc/origin/node/certificates/kubelet-server-current.pem
        - /etc/origin/node/client-ca.crt
        - /etc/origin/node/node.kubeconfig

  tasks:
    - name: make a localhost dir to hold fetched node configs
      delegate_to: localhost
      run_once: true
      file:
        dest: "{{ node_backup_dir }}"
        state: directory

    - name: Create bootstrap.kubeconfig from openshift-infra/node-bootstrapper serviceaccount
      delegate_to: localhost
      run_once: true
      shell: >
        oc serviceaccounts create-kubeconfig node-bootstrapper -n openshift-infra > {{ new_bootstrap }}
      args:
        creates: "{{ new_bootstrap }}"

    - name: Backup node PKI configs
      fetch:
        src: "{{ item }}"
        dest: "{{ node_backup_dir }}"
      with_items: "{{ files_to_replace }}"

    - name: scheduling down
      include_tasks: tasks/drain-node.yml
      static: no

    - name: Remove node PKI configs
      file:
        path: "{{ item }}"
        state: absent
      with_items: "{{ files_to_replace }}"

    - name: Distribute new bootstrap.kubeconfig
      copy:
        src: "{{ new_bootstrap }}"
        dest: /etc/origin/node/
        owner: root
        group: root
        mode: 0600
        backup: yes

    - name: "ALERT - The next step will hang!"
      debug:
        msg: "You must open a second terminal and: 'oc get csr -o name | xargs oc adm certificate approve'"

    - name: Restart node service and generate new CSR
      service:
        name: atomic-openshift-node
        state: restarted


- hosts: localhost
  serial: 1
  gather_facts: no
  connection: local

  tasks:
    - name: Presumably we should pause here for CSR arrival
      pause:
        seconds: 5

    - name: Check for outstanding CSRs
      command: >
        oc get csr -o name
      register: oc_get_csr

    - name: Approve CSRs
      command: >
        oc adm certificate approve {{ item }}
      with_items: "{{ oc_get_csr.stdout_lines }}"


- hosts: primary-nodes,infra-nodes
  serial: 1
  gather_facts: no
 
  tasks:
    - name: Check via master proxy to see if node is healthy after cert generation
      command: >
        oc --config=/etc/origin/master/admin.kubeconfig get --raw /api/v1/nodes/{{inventory_hostname}}/proxy/healthz
      register: node_health
      delegate_to: localhost
      until: node_health.stdout.find("ok")
      delay: 3
      retries: 5

    - name: Display health result
      debug: var=node_health.stdout

    - name: Scheduling node up
      include_tasks: tasks/make-node-schedulable.yml
      static: no
      tags:
        - resume
{% endraw %}{% endhighlight %}

Here are the two tasks which were included above.

{% highlight yaml %}{% raw %}
# tasks/drain-node.yml
# https://kubernetes.io/images/docs/kubectl_drain.svg
# draining a node will make it unschedulable first
- name: drain node of pods
  command: "oc adm drain {{inventory_hostname}} --force --delete-local-data --ignore-daemonsets"
  delegate_to: 127.0.0.1
  when: "'nodes' in group_names"
{% endraw %}{% endhighlight %}

{% highlight yaml %}{% raw %}
# tasks/make-node-schedulable.yml
- name: make node scheduleable
  command: "oc adm manage-node {{inventory_hostname}} --schedulable=true"
  delegate_to: 127.0.0.1
  when: "'nodes' in group_names"
{% endraw %}{% endhighlight %}
