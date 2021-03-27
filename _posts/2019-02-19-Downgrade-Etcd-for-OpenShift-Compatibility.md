---
title: Downgrade Etcd 3.3.11 to 3.2.22 for OpenShift Compatibility
layout: post
tags:
 - etcd
 - openshift
 - OCP3
---

While I was working on [migrating etcd to my master nodes](http://guifreelife.com/blog/2019/02/08/Migration-of-Etcd-to-Masters-for-OpenShift-3.9-Upgrade-to-3.10) I was bitten by an incompatible etcd v3.3.11 RPM made available via RHEL Server Extras repo. Before I got to my last master the RPM was no longer available, and the scaleup playbook failed. I became aware that 3.3.11 [is not compatible](https://access.redhat.com/articles/2176281) and [should not have been made available](https://access.redhat.com/solutions/3885101). 

Unfortunately all members of my etcd cluster were already upgraded and the fix is to take down the cluster, downgrade etcd, and restore from snapshot.  It would be great [if the etcd version was pinned](https://bugzilla.redhat.com/show_bug.cgi?id=1672518) like Docker is.

Here is more or less what I did. YMMV.

**Assumptions:**

- You are running OCP 3.9 with non-containerized etcd
- You have multiple Master nodes
- You have dedicated Etcd nodes
- You are running RHEL, not Atomic nodes
- You upgraded etcd beyond 3.2.x

# Steps

- [ ] **Get endpoint status**

{% highlight bash %}{% raw %}
#!/bin/bash

. /etc/etcd/etcd.conf
. /root/playbook-openshift/openshift-env.sh

ENV=${OPENSHIFT_ENV:-test}
FORMAT="table"
#ENDPOINTS="https://ose-${ENV}-etcd-01.example.com:2379,https://ose-${ENV}-etcd-02.example.com:2379,https://ose-${ENV}-etcd-03.example.com:2379"

ALL_ENDPOINTS=`ETCDCTL_API=3 etcdctl \
        --cert ${ETCD_PEER_CERT_FILE} \
        --key ${ETCD_PEER_KEY_FILE} \
        --cacert ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} \
        --endpoints "$ETCD_ADVERTISE_CLIENT_URLS" \
        --write-out=fields \
        member list | awk '/ClientURL/{printf "%s%s",sep,$3; sep=","}'`

ETCDCTL_API=3 etcdctl \
    --cert ${ETCD_PEER_CERT_FILE} \
    --key ${ETCD_PEER_KEY_FILE} \
    --cacert ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} \
    --endpoints "$ALL_ENDPOINTS" \
    --write-out "$FORMAT" \
    endpoint status
{% endraw %}{% endhighlight %}

{% highlight plain %}{% raw %}
[root@ose-test-etcd-01 bin]# ./etcd-status
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
|          ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://192.0.2.242:2379 | 3cc657644e2e1080 |  3.3.11 |   61 MB |      true |      6997 |  493115674 |
| https://192.0.2.243:2379 | 669fc09764815697 |  3.3.11 |   61 MB |     false |      6997 |  493115674 |
| https://192.0.2.250:2379 | 8268d3895839030d |  3.3.11 |   61 MB |     false |      6997 |  493115674 |
| https://192.0.2.241:2379 | dd1f136e71579ace |  3.3.11 |   61 MB |     false |      6997 |  493115674 |
| https://192.0.2.251:2379 | eafa4cc2f9510e7b |  3.3.11 |   61 MB |     false |      6997 |  493115675 |
+-----------------------------+------------------+---------+---------+-----------+-----------+------------+
{% endraw %}{% endhighlight %}

- [ ] **Create etcd snapshot backup.**

{% highlight bash %}{% raw %}
# https://access.redhat.com/solutions/1981013#comment-1257931
BACKUP_DIR="/var/backup/etcd/$(date +%Y%m%d%H%M)"
mkdir -p ${BACKUP_DIR}/snap

ETCD3="etcdctl --cert ${ETCD_PEER_CERT_FILE} --key ${ETCD_PEER_KEY_FILE} \
        --cacert ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} --endpoints ${ETCD_ADVERTISE_CLIENT_URLS}"

ETCDCTL_API=3 $ETCD3 \
    snapshot save ${BACKUP_DIR}/snap/db
{% endraw %}{% endhighlight %}

> Snapshot saved at /var/backup/etcd/20190219/snap/db

- [ ] **Copy db for some reason. Just cuz**

{% highlight bash %}{% raw %}
[root@ose-test-etcd-01 bin]# cp -p /var/lib/etcd/member/snap/db /tmp/db
{% endraw %}{% endhighlight %}

- [ ] **Distribute saved snapshot `/var/backup/etcd/20190219/snap/db` to all etcd nodes**

{% highlight yaml %}{% raw %}
- hosts: etcd
  vars:
    etcd_backup: /var/backup/etcd/20190219/snap/db
    etcd_backup_copy: 20190219.snap.db

  tasks:
#   - name: create snapshot backup
#     script:
#       name: bin/etcd-backup
#     delegate_to: ose-test-etcd-01.example.com

  - name: download etcd backup from node 1
    fetch:
      src: "{{ etcd_backup }}"
      dest: "./tmp/{{ etcd_backup_copy }}"
      flat: true
    delegate_to: ose-test-etcd-01.example.com
    run_once: true

  - name: distribute etcd backup to all nodes
    copy:
      src: "./tmp/{{ etcd_backup_copy }}"
      dest: "/tmp/{{ etcd_backup_copy }}"
{% endraw %}{% endhighlight %}

- [ ] **Stop etcd on all nodes and rm old data**

{% highlight bash %}{% raw %}
ansible etcd -m service -a 'name=etcd state=stopped'
ansible etcd -m shell -a 'mv /var/lib/etcd{,.deleted}'
{% endraw %}{% endhighlight %}

- [ ] **Downgrade etcd to supported version**

{% highlight bash %}{% raw %}
ansible etcd -m yum -a 'name=etcd-3.2.22 allow_downgrade=yes'
ansible etcd -m file -a 'dest=/var/lib/etcd state=absent'
{% endraw %}{% endhighlight %}

- [ ] **Confirm the version**

{% highlight plain %}{% raw %}
[root@ose-test-master-01 playbook-openshift]# ansible etcd -m command -a 'rpm -q etcd' -o
ose-test-etcd-01.example.com | SUCCESS | rc=0 | (stdout) etcd-3.2.22-1.el7.x86_64
ose-test-etcd-03.example.com | SUCCESS | rc=0 | (stdout) etcd-3.2.22-1.el7.x86_64
ose-test-etcd-02.example.com | SUCCESS | rc=0 | (stdout) etcd-3.2.22-1.el7.x86_64
ose-test-master-02.example.com | SUCCESS | rc=0 | (stdout) etcd-3.2.22-1.el7.x86_64
ose-test-master-01.example.com | SUCCESS | rc=0 | (stdout) etcd-3.2.22-1.el7.x86_64
{% endraw %}{% endhighlight %}

- [ ] **Recover from snapshot on each node**

{% highlight bash %}{% raw %}
#!/bin/bash -x
source /etc/etcd/etcd.conf

# not all etcd nodes will have all members listed in etcd.conf if you used the scaleup playbook
ETCD_INITIAL_CLUSTER=ose-test-etcd-02.example.com=https://192.0.2.242:2380,ose-test-etcd-03.example.com=https://192.0.2.243:2380,ose-test-master-02.example.com=https://192.0.2.250:2380,ose-test-etcd-01.example.com=https://192.0.2.241:2380,ose-test-master-01.example.com=https://192.0.2.251:2380

ETCDCTL_API=3 etcdctl snapshot restore /tmp/20190219.snap.db \
    --name $ETCD_NAME \
    --initial-cluster $ETCD_INITIAL_CLUSTER \
    --initial-cluster-token $ETCD_INITIAL_CLUSTER_TOKEN \
    --initial-advertise-peer-urls $ETCD_INITIAL_ADVERTISE_PEER_URLS \
    --data-dir /var/lib/etcd
{% endraw %}{% endhighlight %}

> eg: ose-test-etcd-01

{% highlight bash %}{% raw %}
  [root@ose-test-etcd-01 ~]# ./restore-etcd.sh
  + . /etc/etcd/etcd.conf
  ++ ETCD_NAME=ose-test-etcd-01.example.com
  ++ ETCD_LISTEN_PEER_URLS=https://192.0.2.241:2380
  ++ ETCD_DATA_DIR=/var/lib/etcd/
  ++ ETCD_HEARTBEAT_INTERVAL=500
  ++ ETCD_ELECTION_TIMEOUT=2500
  ++ ETCD_LISTEN_CLIENT_URLS=https://192.0.2.241:2379
  ++ ETCD_INITIAL_ADVERTISE_PEER_URLS=https://192.0.2.241:2380
  ++ ETCD_INITIAL_CLUSTER=ose-test-etcd-01.example.com=https://192.0.2.241:2380,ose-test-etcd-02.example.com=https://192.0.2.242:2380,ose-test-etcd-03.example.com=https://192.0.2.243:$
  380
  ++ ETCD_INITIAL_CLUSTER_STATE=new
  ++ ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-1
  ++ ETCD_ADVERTISE_CLIENT_URLS=https://192.0.2.241:2379
  ++ ETCD_CA_FILE=/etc/etcd/ca.crt
  ++ ETCD_CERT_FILE=/etc/etcd/server.crt
  ++ ETCD_KEY_FILE=/etc/etcd/server.key
  ++ ETCD_PEER_CA_FILE=/etc/etcd/ca.crt
  ++ ETCD_PEER_CERT_FILE=/etc/etcd/peer.crt
  ++ ETCD_PEER_KEY_FILE=/etc/etcd/peer.key
  + ETCD_INITIAL_CLUSTER=ose-test-etcd-02.example.com=https://192.0.2.242:2380,ose-test-etcd-03.example.com=https://192.0.2.243:2380,ose-test-master-02.example.com=https://192.0.2.250$
  2380,ose-test-etcd-01.example.com=https://192.0.2.241:2380,ose-test-master-01.example.com=https://192.0.2.251:2380
  + SNAPSHOT=/tmp/20190219.snap.db/20190219.snap.db
  + ETCDCTL_API=3
  + etcdctl snapshot restore /tmp/20190219.snap.db/20190219.snap.db --name ose-test-etcd-01.example.com --initial-cluster ose-test-etcd-02.example.com=https://192.0.2.242:2380,ose-
  test-etcd-03.example.com=https://192.0.2.243:2380,ose-test-master-02.example.com=https://192.0.2.250:2380,ose-test-etcd-01.example.com=https://192.0.2.241:2380,ose-test-master-01.pix
  ar.com=https://192.0.2.251:2380 --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls https://192.0.2.241:2380 --data-dir /var/lib/etcd
  2019-02-19 11:33:09.696902 I | mvcc: restore compact to 196581339
  2019-02-19 11:33:09.723669 I | etcdserver/membership: added member 3beb6ea7fc78a5f7 [https://192.0.2.251:2380] to cluster f09fa88ac65802b7
  2019-02-19 11:33:09.723698 I | etcdserver/membership: added member c80ffa9b8823e61d [https://192.0.2.250:2380] to cluster f09fa88ac65802b7
  2019-02-19 11:33:09.723709 I | etcdserver/membership: added member dd1f136e71579ace [https://192.0.2.241:2380] to cluster f09fa88ac65802b7
  2019-02-19 11:33:09.723719 I | etcdserver/membership: added member f54da589b60c16be [https://192.0.2.243:2380] to cluster f09fa88ac65802b7
  2019-02-19 11:33:09.723732 I | etcdserver/membership: added member f573d8d196b92c82 [https://192.0.2.242:2380] to cluster f09fa88ac65802b7
{% endraw %}{% endhighlight %}

- [ ] **Repeat above restore on all etcd nodes**

- [ ] **Fix etcd permissions**

{% highlight bash %}{% raw %}
ansible etcd -m command -a 'chown etcd:etcd -R /var/lib/etcd'
ansible etcd -m command -a 'restorecon -Rv /var/lib/etcd'
{% endraw %}{% endhighlight %}

- [ ] **Start etcd and check status.**

{% highlight bash %}{% raw %}
ansible etcd -m service -a 'name=etcd state=started'
{% endraw %}{% endhighlight %}

> eg:

{% highlight bash %}{% raw %}
[root@ose-test-etcd-01 bin]# ./etcd-status
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
|          ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://192.0.2.251:2379 | 3beb6ea7fc78a5f7 |  3.2.22 |   61 MB |     false |        15 |      11113 |
| https://192.0.2.250:2379 | c80ffa9b8823e61d |  3.2.22 |   61 MB |     false |        15 |      11113 |
| https://192.0.2.241:2379 | dd1f136e71579ace |  3.2.22 |   61 MB |      true |        15 |      11113 |
| https://192.0.2.243:2379 | f54da589b60c16be |  3.2.22 |   61 MB |     false |        15 |      11113 |
| https://192.0.2.242:2379 | f573d8d196b92c82 |  3.2.22 |   61 MB |     false |        15 |      11113 |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
{% endraw %}{% endhighlight %}

- [ ] **Restart master api**

{% highlight bash %}{% raw %}
ansible masters -m service -a 'name=atomic-openshift-master-api state=restarted'
{% endraw %}{% endhighlight %}

- [ ] **I messed up and had to start over, so here's a more compact summary of restore using the same snapshot.**

{% highlight bash %}{% raw %}
ansible etcd -m service -a 'name=etcd state=stopped'
ansible etcd -m file -a 'dest=/var/lib/etcd state=absent'
ansible etcd -m copy -a 'src=bin/restore-etcd.sh dest=/root/restore-etcd.sh mode=0755'

# login to each node, starting with etcd01 where ETCD_INITIAL_CLUSTER_STATE=new and run restore
/root/restore-etcd.sh

# fix perms
ansible etcd -m command -a 'chown etcd:etcd -R /var/lib/etcd'
ansible etcd -m command -a 'restorecon -Rv /var/lib/etcd'
# start etcd
ansible etcd -m service -a 'name=etcd state=started'
# check that the state from each node matches
ansible etcd -m script -a 'bin/etcd-status'
ansible masters -m service -a 'name=atomic-openshift-master-api state=restarted'
{% endraw %}{% endhighlight %}

