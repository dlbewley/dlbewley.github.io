---
title: Migration of Etcd to Masters for OpenShift 3.9 to 3.10 Upgrade
layout: post
tags:
 - etcd
 - openshift
 - OCP3.9
 - OCP3.10
---

As of OpenShift Container Platform 3.10 etcd is expected to run in [static pods](https://docs.openshift.com/container-platform/3.10/release_notes/ocp_3_10_release_notes.html#ocp-310-control-plane-changes) on the master nodes in the control plane. You may have a deployed an HA cluster with dedicated etcd nodes managed with systemd. How do you migrate the this new architecture?

**Assumptions:**

- You are running OCP 3.9
- You have multiple Master nodes
- You have dedicated Etcd nodes
- You are running RHEL, not Atomic nodes

**Outline:**

- Backup etcd
- Scale up Etcd cluster to include Master nodes
- Configure Openshift Masters to ignore the old Etcd nodes
- Scale down etcd cluster to remove old Etcd nodes

# Detailed Steps #

Follow along in this document [https://docs.openshift.com/container-platform/3.9/admin_guide/assembly_replace-etcd-member.html](https://docs.openshift.com/container-platform/3.9/admin_guide/assembly_replace-etcd-member.html)
You may find [some etcd aliases handy](http://guifreelife.com/blog/2019/02/08/Etcd-Shortcut-With-Peer-Auth) before proceeding.

- [ ] **Create an `new_etcd` ansible group in your inventory file.**

- [ ] **Add the first Master node to this `new_etcd` group for testing.**

- [ ] **Add `new_etcd` group as a child to the `OSEv3` ansible group.**

- [ ] **Confirm your cluster health on the first etcd server.**

{% highlight bash %}{% raw %}
#!/bin/bash

. /etc/etcd/etcd.conf

ENV=${OPENSHIFT_ENV:-test}
ENDPOINTS="https://ose-${ENV}-etcd-01.example.com:2379,https://ose-${ENV}-etcd-02.example.com:2379,https://ose-${ENV}-etcd-03.example.com:2379"

ETCDCTL_API=3 etcdctl \
    --cert ${ETCD_PEER_CERT_FILE} \
    --key ${ETCD_PEER_KEY_FILE} \
    --cacert ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} \
    --endpoints "$ENDPOINTS" \
    endpoint health
{% endraw %}{% endhighlight %}

{% highlight plain %}{% raw %}
[root@ose-test-etcd-01 bin]# ./etcd-health
https://ose-test-etcd-01.example.com:2379 is healthy: successfully committed proposal: took = 2.41743ms
https://ose-test-etcd-03.example.com:2379 is healthy: successfully committed proposal: took = 2.363286ms
https://ose-test-etcd-02.example.com:2379 is healthy: successfully committed proposal: took = 2.213456ms
{% endraw %}{% endhighlight %}


- [ ] **[Create a backup of your etcd](https://docs.openshift.com/container-platform/3.9/day_two_guide/environment_backup.html#backing-up-etcd_environment-backup) data and configuration.**

> Because of the [migration during the upgrade](https://docs.openshift.com/container-platform/3.7/upgrading/migrating_etcd.html) to 3.7, I am assuming I do not need to back up v2 data. That is [somewhat TBD](https://lists.openshift.redhat.com/openshift-archives/users/2019-February/msg00010.html), however.

  - [OpenShift-Users mailing list post](https://lists.openshift.redhat.com/openshift-archives/users/2019-February/msg00029.html)
  - [BZ 1579304](https://bugzilla.redhat.com/show_bug.cgi?id=1579304)
  - [BZ 1656397](https://bugzilla.redhat.com/show_bug.cgi?id=1656397)


{% highlight bash %}{% raw %}
#!/bin/bash
# https://docs.openshift.com/container-platform/latest/admin_guide/backup_restore.html
# https://access.redhat.com/solutions/1981013#comment-1257931

. /etc/etcd/etcd.conf

ETCD3="etcdctl --cert ${ETCD_PEER_CERT_FILE} \
  --key ${ETCD_PEER_KEY_FILE} \
  --cacert ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} \
  --endpoints ${ETCD_ADVERTISE_CLIENT_URLS}"

BACKUP_DIR="/var/backup/etcd/$(date +%Y%m%d%H%M)"
mkdir -p ${BACKUP_DIR}/snap
cp -rp /etc/etcd "${BACKUP_DIR}/"
cp -p $0 "${BACKUP_DIR}/"

ETCDCTL_API=3 $ETCD3 \
        snapshot save ${BACKUP_DIR}/snap/db

# Restore:
# . ${BACKUP_DIR}/etcd/etcd.conf
# ETCDCTL_API=3 $ETCD3 \
#    --name $ETCD_NAME \
#    --initial-cluster $ETCD_INITIAL_CLUSTER \
#    --initial-cluster-token $ETCD_INITIAL_CLUSTER_TOKEN \
#    --initial-advertise-peer-urls $ETCD_INITIAL_ADVERTISE_PEER_URLS \
#    snapshot restore ${BACKUP_DIR}/snap/db
{% endraw %}{% endhighlight %}


- [ ] **Run the [etcd scaleup playbook](https://github.com/openshift/openshift-ansible/blob/release-3.9/playbooks/openshift-etcd/scaleup.yml)**

{% highlight bash %}{% raw %}
#!/bin/bash
# https://docs.openshift.com/container-platform/3.9/admin_guide/assembly_replace-etcd-member.html
PLAYBOOK=/usr/share/ansible/openshift-ansible/playbooks/openshift-etcd/scaleup.yml

ansible-playbook -vvv \
        -i hosts "$PLAYBOOK" \
        | tee $(date +%Y%m%d-%H%M)-etcd-scaleup.log
{% endraw %}{% endhighlight %}


> In my case I found etcd had been accidentally started by hand with a default config file which listened on localhost. The config file was [modified by the etcd role](https://github.com/openshift/openshift-ansible/blob/release-3.9/roles/etcd/tasks/main.yml#L119) and the restart etcd handler was notified, but it was skipped.  This caused the [etcd cluster status check task](https://github.com/openshift/openshift-ansible/blob/release-3.9/playbooks/openshift-etcd/private/scaleup.yml#L51) to timeout, and subsequent steps in the playbook to fail.
>
> After restarting etcd at 18:43 the cluster reports as healthy, and I re-ran the playbook successfully.
>

After the playbook has been run successfuly it can be seen that the master node has been added as an etcd endpoint in `/etc/origin/master/master-config.yaml` on every master node.

{% highlight yaml %}{% raw %}
etcdClientInfo:
  ca: master.etcd-ca.crt
  certFile: master.etcd-client.crt
  keyFile: master.etcd-client.key
  urls:
  - https://ose-test-etcd-01.example.com:2379
  - https://ose-test-etcd-02.example.com:2379
  - https://ose-test-etcd-03.example.com:2379
  - https://ose-test-master-01.example.com:2379
{% endraw %}{% endhighlight %}


- [ ] **This master is done. Move this first master from the `new_etcd` to the `etcd` ansible group.** _Leave it in any other groups it is already a member of of course._

- [ ] **Remove old `ose-test-etcd-03` node from the `etcd` ansible group.**

- [ ] **Update the `master-config.yaml` to include only the hosts remaining in the `etcd` ansible group and restart api service.**

I considered the [`modify_yaml`](https://github.com/openshift/openshift-ansible/blob/release-3.9/roles/lib_utils/library/modify_yaml.py) but after noticing it inserted some `nulls` and converted some doule quotes to single quotes, I was happy to find the [`yedit`](https://github.com/openshift/openshift-ansible/blob/release-3.9/roles/lib_utils/library/yedit.py) module.


{% highlight yaml %}{% raw %}
---
# playbook to replace currently configured master etcd URLs with
# the hosts found in ansible etcd group
- hosts: masters

  vars:
    openshift_master_fire_handlers: true

  roles:
    # https://github.com/openshift/openshift-ansible/tree/release-3.9/roles/lib_utils/library
    - lib_utils
    - openshift_facts

  tasks:

    - name: Gather Cluster facts
      openshift_facts:
        role: common

    - name: Derive etcd url list
      set_fact:
        openshift_master_etcd_urls: "{{ groups['etcd'] | lib_utils_oo_etcd_host_urls(l_use_ssl, openshift_master_etcd_port) }}"
      vars:
        l_use_ssl: "{{ openshift_master_etcd_use_ssl | default(True) | bool}}"
        openshift_master_etcd_port: "{{ etcd_client_port | default('2379') }}"

    - name: Configure ectcd url list
      yedit:
        src: "{{ openshift.common.config_base }}/master/master-config.yaml"
        key: etcdClientInfo.urls
        value: "{{ openshift_master_etcd_urls }}"
        backup: yes
      notify: restart master api

  handlers:
    # https://github.com/openshift/openshift-ansible/blob/release-3.9/roles/openshift_master/handlers/main.yml
    - import_tasks: /usr/share/ansible/openshift-ansible/roles/openshift_master/handlers/main.yml
{% endraw %}{% endhighlight %}


- [ ] **Verify OpenShift operation**

- [ ] **Remove old `ose-test-etcd-03` node from etcd cluster.**

  {% highlight yaml %}{% raw %}
  [root@ose-test-master-01 etcd]# etcdctl3 member list
  3cc657644e2e1080, started, ose-test-etcd-02.example.com, https://192.0.2.242:2380, https://192.0.2.242:2379
  669fc09764815697, started, ose-test-etcd-03.example.com, https://192.0.2.243:2380, https://192.0.2.243:2379
  dd1f136e71579ace, started, ose-test-etcd-01.example.com, https://192.0.2.241:2380, https://192.0.2.241:2379
  eafa4cc2f9510e7b, started, ose-test-master-01.example.com, https://192.0.2.251:2380, https://192.0.2.251:2379

  [root@ose-test-master-01 etcd]# etcdctl3 member remove 669fc09764815697
  {% endraw %}{% endhighlight %}


- [ ] **Repeat for Masters 2 and 3 and etcd nodes 2 and 1.**

You are now one step closer to OpenShift 3.10.


At this point etcd should be running only on the 3 Master nodes and not on the old Etcd nodes. All the masters should know this, and you are one step closer to being able to upgrade to OpenShift 3.10.

# Problems

**Solved**

- As I mentioned I had accidentally started etcd with a default config and the scaleup playbook did not expect this condition.

**Lingering**

- I scaled up 2 masters as etcd nodes they got etcd 3.3.11 installed and when I went to scale up the 3rd master soon after suddenly the newest etcd RPM was 3.2.22. Guess what. Those are not compatible. In fact OpenShift is not certified to work with etcd 3.3. Etcd 3.3 should be excluded in `yum.conf` but it is not [BZ 1672518](https://bugzilla.redhat.com/show_bug.cgi?id=1672518)! Oh, and it gets worse!  This KB points out a 3.2 etcd container image got a 3.3 etcd binary into it somehow! "[ETCD hosts were upgraded to version 3.3.11.]( https://access.redhat.com/solutions/3885101)".  Wow. WTF?

# See Also

- [Backup and Restore in OpenShift Container Platform 3](https://access.redhat.com/solutions/1981013)
- [Replacing a failed etcd member](https://docs.openshift.com/container-platform/3.9/admin_guide/assembly_replace-etcd-member.html)
- [Known Issues when upgrading to OpenShift 3.10](https://access.redhat.com/solutions/3631141)
- [Role for configuring master config](https://github.com/canit00/role_cluster_config)
- [ETCD hosts were upgraded to version 3.3.11.](https://access.redhat.com/solutions/3885101)
- [Etcd 3.3 should be excluded but it is not BZ 1672518](https://bugzilla.redhat.com/show_bug.cgi?id=1672518)
