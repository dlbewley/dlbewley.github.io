---
title: Changing the SSL Certificate for OpenShift Console
layout: post
tags:
 - kubernetes
 - openshift
 - OSE3.1
 - ssl
---

OpenShift has an internal CA for generating certificates to authenticate intra-cluster communication, but your browser doesn't trust this CA. Perhaps you want to fix that without mucking with the internal SSL communication? I did. Here is how.

This [OpenShift doc](https://docs.openshift.org/latest/install_config/certificate_customization.html) _explains_ how to do this, but it isn't very clear, to me at least.


# Overview #

An outline of the steps:

- Only make changes to the public URLs and not any internal URLs.
- Create a `namedCertificates` section in both `/servingInfo` and `/assetConfig/servingInfo` sections of `/etc/origin/master/master-config.yaml`.
- In those repeated sections:
  - identify a certificate and key
  - identify the hostname(s) to match with that cert/key pair

Your installation may include the following hosts:

Name                         | IP
-----------------------------|------------
ose-ha-master-01.example.com | 192.0.2.21
ose-ha-master-02.example.com | 192.0.2.22
ose-ha-master-03.example.com | 192.0.2.23
ose-ha-lb-01.example.com     | 192.0.2.41
master.os.example.com        | _CNAME ose-ha-lb-01.example.com_
openshift.example.com        | _CNAME ose-ha-lb-01.example.com_

In this case `openshift.example.com` is an alias to the loadbalancer which directs traffic back to the 3 masters. The load balancer passes the traffic through for TLS termination on port 8443 of the master servers. Therefore, all three masters need to be updated.

# Install Your Certificate #

Copy your certificate and key to the masters in the `/etc/origin/master` directory and give them the following names.

- `wildcard.example.com.crt`
- `wildcard.example.com.key`

Update `/etc/origin/master/master-config.yaml` to reference those certificates when accessing the public master URL.

**Before**

```yaml
apiLevels:
- v1
apiVersion: v1
assetConfig:
  logoutURL: ""
  masterPublicURL: https://openshift.example.com:8443
  publicURL: https://openshift.example.com:8443/console/
  servingInfo:
    bindAddress: 0.0.0.0:8443
    bindNetwork: tcp4
    certFile: master.server.crt
    clientCA: ""
    keyFile: master.server.key
    maxRequestsInFlight: 0
    requestTimeoutSeconds: 0
  metricsPublicURL: "https://metrics.os.example.com/hawkular/metrics"
#... skip to the botom
servingInfo:
  bindAddress: 0.0.0.0:8443
  bindNetwork: tcp4
  certFile: master.server.crt
  clientCA: ca.crt
  keyFile: master.server.key
  maxRequestsInFlight: 500
  requestTimeoutSeconds: 3600
```

**After**

```yaml
apiLevels:
- v1
apiVersion: v1
assetConfig:
  logoutURL: ""
  masterPublicURL: https://openshift.example.com:8443
  publicURL: https://openshift.example.com:8443/console/
  servingInfo:
    bindAddress: 0.0.0.0:8443
    bindNetwork: tcp4
    certFile: master.server.crt
    clientCA: ""
    keyFile: master.server.key
    maxRequestsInFlight: 0
    requestTimeoutSeconds: 0
    namedCertificates:
      - certFile: wildcard.example.com.crt
        keyFile: wildcard.example.com.key
        names:
          - "openshift.example.com"
  metricsPublicURL: "https://metrics.os.example.com/hawkular/metrics"
#... skip to the botom
servingInfo:
  bindAddress: 0.0.0.0:8443
  bindNetwork: tcp4
  certFile: master.server.crt
  clientCA: ca.crt
  keyFile: master.server.key
  maxRequestsInFlight: 500
  requestTimeoutSeconds: 3600
  namedCertificates:
    - certFile: wildcard.example.com.crt
      keyFile: wildcard.example.com.key
      names:
        - "openshift.example.com"
```

Restart the master and watch for TLS errors for a few seconds. You will notice problems pretty quickly if you affected the internal API URL.

```bash
systemctl restart atomic-openshift-master-api
journalctl -f
```

# Using Ansible #

There is support for this in [the playbook](https://github.com/openshift/openshift-ansible), which is probably the best method, but **I have not tested this yet**.

Update your inventory for [OpenShift Advanced Installation](https://docs.openshift.com/enterprise/3.1/install_config/install/advanced_install.html#advanced-install-custom-certificates) while referring to the [byo example](https://github.com/openshift/openshift-ansible/blob/master/inventory/byo/hosts.ose.example#L180).

```ini
openshift_master_cluster_method=native
openshift_master_cluster_hostname=master.os.example.com
openshift_master_cluster_public_hostname=openshift.example.com
openshift_master_overwrite_named_certificates: true
#
# Provide local certificate paths which will be deployed to masters
openshift_master_named_certificates=[{"certfile": "wildcard.example.com.crt", "keyfile": "wildcard.example.com.key"}]
#
# Detected names may be overridden by specifying the "names" key
#openshift_master_named_certificates=[{"certfile": "wildcard.example.com.crt", "keyfile": "wildcard.example.com.key", "names": ["openshift.example.com"]}]
```

# Related Reading #

- [OpenShift Advanced Installation](https://docs.openshift.com/enterprise/3.1/install_config/install/advanced_install.html#configuring-ansible)
- [OpenShift Certificate Customization](https://docs.openshift.org/latest/install_config/certificate_customization.html)
- Trello Card [Allow setting custom serving certs for specific hostnames in API / Console using SNI](https://trello.com/c/Gc3FDSK8)
- [PR5097](https://github.com/openshift/origin/pull/5097)
- [Run OpenShift console on port 443](http://akrambenaissi.com/2016/02/21/run-openshift-console-on-port-443/)
- [Make OpenShift console available on port 443](https://alword.wordpress.com/2016/03/11/make-openshift-console-available-on-port-443-https/)
