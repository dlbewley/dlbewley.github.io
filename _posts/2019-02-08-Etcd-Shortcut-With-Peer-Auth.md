---
title: Etcdctl v2 and v3 Aliases for Peer Authenticated Commands
layout: post
tags:
 - etcd
 - openshift
---

Getting all the arguments to `etcdctl` right can be a bit of a pain. Here are a couple of aliases which take advantage of the values in the `etcd.conf` file.

{% highlight bash %}{% raw %}
alias etcd2='. /etc/etcd/etcd.conf && \
    ETCDCTL_API=2 etcdctl \
    --cert-file ${ETCD_PEER_CERT_FILE} \
    --key-file ${ETCD_PEER_KEY_FILE} \
    --ca-file ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} \
    --endpoints "${ETCD_ADVERTISE_CLIENT_URLS}"'

alias etcd3='. /etc/etcd/etcd.conf && \
    ETCDCTL_API=3 etcdctl \
    --cert ${ETCD_PEER_CERT_FILE} \
    --key ${ETCD_PEER_KEY_FILE} \
    --cacert ${ETCD_PEER_TRUSTED_CA_FILE:-$ETCD_PEER_CA_FILE} \
    --endpoints "${ETCD_ADVERTISE_CLIENT_URLS}"'
{% endraw %}{% endhighlight %}

If you are using OpenShift, you may also find that you already have some bash functions [enabled by the etcd role](https://github.com/openshift/openshift-ansible/blob/master/roles/etcd/tasks/drop_etcdctl.yml) in `/etc/profile.d/etcdctl.sh`. They will look different depending on your version. Below is from 3.9.

{% highlight bash %}{% raw %}
#!/bin/bash
# Sets up handy aliases for etcd, need etcdctl2 and etcdctl3 because
# command flags are different between the two. Should work on stand
# alone etcd hosts and master + etcd hosts too because we use the peer keys.
etcdctl2() {
 /usr/bin/etcdctl --cert-file /etc/etcd/peer.crt --key-file /etc/etcd/peer.key --ca-file /etc/etcd/ca.crt -C https://`hostname`:2379 ${@}

}

etcdctl3() {
 ETCDCTL_API=3 /usr/bin/etcdctl --cert /etc/etcd/peer.crt --key /etc/etcd/peer.key --cacert /etc/etcd/ca.crt --endpoints https://`hostname`:2379 ${@}
}
{% endraw %}{% endhighlight %}

**Example:**

{% highlight bash %}{% raw %}
[root@ose-test-etcd-01 bin]# etcd3 member list -w table
+------------------+---------+--------------------------------+--------------------------+--------------------------+
|        ID        | STATUS  |             NAME               |         PEER ADDRS       |        CLIENT ADDRS      |
+------------------+---------+--------------------------------+--------------------------+--------------------------+
| 3cc657644e2e1080 | started |   ose-test-etcd-02.example.com | https://192.0.2.242:2380 | https://192.0.2.242:2379 |
| 669fc09764815697 | started |   ose-test-etcd-03.example.com | https://192.0.2.243:2380 | https://192.0.2.243:2379 |
| dd1f136e71579ace | started |   ose-test-etcd-01.example.com | https://192.0.2.241:2380 | https://192.0.2.241:2379 |
+------------------+---------+--------------------------------+--------------------------+--------------------------+
{% endraw %}{% endhighlight %}

{% highlight bash %}{% raw %}
[root@ose-test-etcd-01 bin]# etcd2 cluster-health
member 3cc657644e2e1080 is healthy: got healthy result from https://192.0.2.242:2379
member 669fc09764815697 is healthy: got healthy result from https://192.0.2.243:2379
member dd1f136e71579ace is healthy: got healthy result from https://192.0.2.241:2379
cluster is healthy
{% endraw %}{% endhighlight %}
