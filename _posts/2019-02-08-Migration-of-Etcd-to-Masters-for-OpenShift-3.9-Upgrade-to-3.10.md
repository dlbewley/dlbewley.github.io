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

_This is WIP!_

**Assumptions:**

- You are running OCP 3.9
- You have multiple Master nodes
- You have dedicated Etcd nodes
- You are running RHEL, not Atomic nodes

**Outline:**

- Backup etcd
- Scale up etcd to include Master nodes
- Configure cluster to use new etcd endpoints
- Scale down etcd to remove Etcd nodes

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


> In my case I found etcd had been started erroneously at 16:57 with a default config file which listened on localhost. The config file was [modified by the etcd role](https://github.com/openshift/openshift-ansible/blob/release-3.9/roles/etcd/tasks/main.yml#L119) at 17:36 and the restart etcd handler was notified but it was skipped.  This caused the [cluster status check task](https://github.com/openshift/openshift-ansible/blob/release-3.9/playbooks/openshift-etcd/private/scaleup.yml#L51) to timeout, and subsequent steps in the playbook to fail.

  {% highlight plain %}{% raw %}
  [root@ose-test-master-01 3.9.60]# grep -E '(TASK|HAND)' 20190207-1735-etcd-scaleup.log | tail
  TASK [etcd : Install Etcd system container] *************************************************
  TASK [etcd : Validate permissions on the config dir] ****************************************
  TASK [etcd : Write etcd global config file] *************************************************
  NOTIFIED HANDLER restart etcd
  TASK [etcd : Enable etcd] *******************************************************************
  TASK [etcd : Set fact etcd_service_status_changed] ******************************************
  TASK [nickhammond.logrotate : nickhammond.logrotate | Install logrotate] ********************
  TASK [nickhammond.logrotate : nickhammond.logrotate | Setup logrotate.d scripts] ************
  RUNNING HANDLER [etcd : restart etcd] *******************************************************
  TASK [Verify cluster is stable] *************************************************************
  {% endraw %}{% endhighlight %}

  {% highlight plain %}{% raw %}
  RUNNING HANDLER [etcd : restart etcd] *******************************************************
  skipping: [ose-test-master-01.example.com] => {
      "changed": false,
      "skip_reason": "Conditional result was False",
      "skipped": true
  }
  META: ran handlers
  {% endraw %}{% endhighlight %}

  {% highlight plain %}{% raw %}
  PLAY RECAP **********************************************************************************
  localhost                  : ok=26   changed=0    unreachable=0    failed=0
  ose-test-etcd-01.example.com : ok=23   changed=0    unreachable=0    failed=0
  ose-test-etcd-02.example.com : ok=20   changed=0    unreachable=0    failed=0
  ose-test-etcd-03.example.com : ok=20   changed=0    unreachable=0    failed=0
  ose-test-lb-01.example.com   : ok=17   changed=0    unreachable=0    failed=0
  ose-test-master-01.example.com : ok=83   changed=19   unreachable=0    failed=1
  ose-test-master-02.example.com : ok=21   changed=0    unreachable=0    failed=0
  ose-test-node-01.example.com : ok=0    changed=0    unreachable=0    failed=0
  ose-test-node-02.example.com : ok=0    changed=0    unreachable=0    failed=0
  ose-test-node-03.example.com : ok=0    changed=0    unreachable=0    failed=0
  ose-test-node-04.example.com : ok=0    changed=0    unreachable=0    failed=0
  {% endraw %}{% endhighlight %}

  {% highlight plain %}{% raw %}
  [root@ose-test-master-01 etcd]# systemctl status etcd -l
  ● etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2019-02-07 16:57:27 PST; 56min ago
  Main PID: 83742 (etcd)
    Tasks: 16
   Memory: 35.9M
   CGroup: /system.slice/etcd.service
           └─83742 /usr/bin/etcd --name=default --data-dir=/var/lib/etcd/default.etcd --listen-client-urls=http://localhost:2379
  {% endraw %}{% endhighlight %}

  {% highlight plain %}{% raw %}
  [root@ose-test-master-01 etcd]# ls -l /etc/etcd/etcd.conf*
  -rw-r--r--. 1 root root 1634 Feb  7 17:36 /etc/etcd/etcd.conf
  -rw-r--r--. 1 root root 1686 Jan 14 05:02 /etc/etcd/etcd.conf.34879.2019-02-07@17:36:37~
  {% endraw %}{% endhighlight %}

> After restarting etcd at 18:43 the cluster reports as healthy.

- [ ] **Re-run scaleup playbook allowing it to complete now that above has been rectified.**

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


- [ ] **This master is done. Move this first master from the `new_etcd` to the `etcd` ansible group.** Leave it in any other groups it is already a member of of course.

- [ ] **Remove an old etcd node eg. `ose-test-etcd-03` from `master-config.yaml`**

> Consider the [modify_yaml module](https://github.com/openshift/openshift-ansible/blob/release-3.9/roles/lib_utils/library/modify_yaml.py) and [canit00 role](https://github.com/canit00/role_cluster_config) if that needs to be 

- [ ] **Restart API on masters `systemctl restart atomic-openshift-master-api`**

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

- [ ] **Take a warm bath.**


At this point etcd should be running only on the 3 Master nodes and not on the old Etcd nodes. All the masters should know this, and you are one step closer to being able to upgrade to OpenShift 3.10.


# See Also

- [Backup and Restore in OpenShift Container Platform 3](https://access.redhat.com/solutions/1981013)
- [Replacing a failed etcd member](https://docs.openshift.com/container-platform/3.9/admin_guide/assembly_replace-etcd-member.html)
- [Known Issues when upgrading to OpenShift 3.10](https://access.redhat.com/solutions/3631141)
- [Role for configuring master config](https://github.com/canit00/role_cluster_config)

