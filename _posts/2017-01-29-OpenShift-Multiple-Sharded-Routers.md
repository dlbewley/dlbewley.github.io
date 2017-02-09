---
title: Configuring OpenShift with Multiple Sharded Routers
layout: post
tags:
 - haproxy
 - kubernetes
 - OCP3.3
 - openshift
 - router
---

I needed to host a service that would be consumed by a [closed client](http://www.teradici.com/) that insists on speaking HTTPS on port 50,000. To solve this, I added a 2nd router deployment and used the OpenShift router sharding feature to selectively enable routes on the 2nd router by way of selectors.

To summarize:

*[Existing HA router](/blog/2016/03/01/OpenShift-3-HA-Routing):*

- HTTP 80
- HTTPS 443
- Haproxy Stats 1,936

*Added HA router:*

- HTTP 49,999
- HTTPS 50,000
- Haproxy Stats 51,936


# How To #

## Open infra node firewalls ##

- Open firewall on infra nodes where router will run to allow new http and https port

```bash
 iptables -A OS_FIREWALL_ALLOW -m tcp -p tcp --dport 49999 -j ACCEPT
 iptables -A OS_FIREWALL_ALLOW -m tcp -p tcp --dport 50000 -j ACCEPT
```

- This can also be done with Ansible and the [os_firewall role](https://github.com/openshift/openshift-ansible/tree/master/roles/os_firewall) in your playbook. (_untested_)

```yaml
- hosts: infra-nodes

  vars:
    os_firewall_use_firewalld: False
    os_firewall_allow:
      - service: teradici-http
        port: 49999/tcp
      - service: teradici-https
        port: 50000/tcp

  roles:
    - os_firewall
```

## Create a router ##

- Create a router called _ha-router-teradici_ with `oa adm router` or `oadm router` on these ports and also make sure the stats port does not clash with existing router on port 1936

```bash
[root@ose-test-master-01 ~]# oc get nodes --show-labels
NAME                           STATUS    AGE       LABELS
ose-test-master-01.example.com   Ready     180d      kubernetes.io/hostname=ose-test-master-01.example.com,region=master,zone=rhev
ose-test-master-02.example.com   Ready     180d      kubernetes.io/hostname=ose-test-master-02.example.com,region=master,zone=rhev
ose-test-node-01.example.com     Ready     180d      ha-router=primary,kubernetes.io/hostname=ose-test-node-01.example.com,region=infra,zone=rhev
ose-test-node-02.example.com     Ready     180d      ha-router=primary,kubernetes.io/hostname=ose-test-node-02.example.com,region=infra,zone=rhev
ose-test-node-03.example.com     Ready     180d      kubernetes.io/hostname=ose-test-node-03.example.com,region=primary,zone=rhev
ose-test-node-04.example.com     Ready     180d      kubernetes.io/hostname=ose-test-node-04.example.com,region=primary,zone=rhev

[root@ose-test-master-01 ~]#  oadm router ha-router-teradici \
    --ports='49999:49999,50000:50000' \
    --stats-port=51936 \
    --replicas=2 \
    --selector="ha-router=primary" \
    --selector="region=infra" \
    --labels="ha-router=teradici" \
    --default-cert=201602_router_wildcard.os.example.com.pem \
    --service-account=router
```

_GOOD_: I see that the ports are set properly in the haproxy.config and the service objects

```bash
[root@ose-test-master-01 ~]# oc get all -l ha-router=teradici
NAME                            REVISION        DESIRED       CURRENT   TRIGGERED BY
dc/ha-router-teradici           1               2             2         config
NAME                            DESIRED         CURRENT       AGE
rc/ha-router-teradici-1         2               2             15m
NAME                            CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
svc/ha-router-teradici          172.30.207.88   <none>        49999/TCP,50000/TCP,51936/TCP   15m
NAME                            READY           STATUS        RESTARTS                        AGE
po/ha-router-teradici-1-3ctx5   1/1             Running       0                               14m
po/ha-router-teradici-1-w8plh   1/1             Running       0                               14m
```

_BAD_: However,  the env in the DC is partially incorrect:

```bash
[root@ose-test-master-01 ~]# oc get dc ha-router-teradici -o json \
  | jq -c '. | .spec.template.spec.containers[].env[] | select(.name | contains("PORT")) '
{"name":"ROUTER_SERVICE_HTTPS_PORT","value":"443"}
{"name":"ROUTER_SERVICE_HTTP_PORT","value":"80"}
{"name":"STATS_PORT","value":"51936"}
```

I think that should be considered a bug in the `oadm router command`.

The haproxy template uses those values to bind ports like this:

```bash
$ docker run --rm --interactive=true --tty --entrypoint=cat \
        registry.access.redhat.com/openshift3/ose-haproxy-router:v3.3.1.7 haproxy-config.template | grep -B1 'bind '
frontend public
  bind :{{env "ROUTER_SERVICE_HTTP_PORT" "80"}}
--
  frontend public_ssl
  bind :{{env "ROUTER_SERVICE_HTTPS_PORT" "443"}}
--
    # terminate ssl on edge
  bind 127.0.0.1:{{env "ROUTER_SERVICE_SNI_PORT" "10444"}} ssl no-sslv3 {{ if (len .DefaultCertificate) gt 0 }}crt {{.DefaultCertificate}}{{ else }}crt /var/lib/haproxy/conf/default_pub_keys.pem{{ end }} crt {{ $workingDir }}/certs accept-proxy
--
    # terminate ssl on edge
  bind 127.0.0.1:{{env "ROUTER_SERVICE_NO_SNI_PORT" "10443"}} ssl no-sslv3 {{ if (len .DefaultCertificate) gt 0 }}crt {{.DefaultCertificate}}{{ else }}crt /var/lib/haproxy/conf/default_pub_keys.pem{{ end }} accept-proxy
```

## Fix the router deploymentconfig ##

- Fix port number environment variables in the new router deploymentconfig

```bash
[root@ose-test-master-01 ~]# oc set env dc/ha-router-teradici  ROUTER_SERVICE_HTTP_PORT=49999
[root@ose-test-master-01 ~]# oc set env dc/ha-router-teradici  ROUTER_SERVICE_HTTPS_PORT=50000
```

_BAD_: Docker inspect still shows me the container is on port 80 and 443. That is because of the hardcoded `EXPOSE` directive in the Dockerfile. Based on the ENV this seems to be a non-issue.

```bash
[root@ose-test-node-01 ~]# docker inspect --format='{{.Config.ExposedPorts}}'  dd02066d4845
map[443/tcp:{} 53/tcp:{} 80/tcp:{} 8443/tcp:{}]

[root@ose-test-master-01 ~]# docker run --rm --interactive=true --tty --entrypoint=grep \
  registry.access.redhat.com/openshift3/ose-haproxy-router:v3.3.1.7 \
  EXPOSE /var/lib/haproxy/Dockerfile-openshift3-ose-haproxy-router-v3.3.1.7-0
EXPOSE 80 443
```

## Add labels for sharding ##

- Add `ROUTE_LABLES` env var to the router deployment config for sharding

```
[root@ose-test-master-01 ~]# oc set env dc/ha-router-teradici  ROUTE_LABELS="router=teradici"
```

- Add matching label to an existing route to make it available on new router

```
[root@ose-test-master-01 ~]# oc label route v3simplebottle -n user1 router=teradici
```

I did not preclude the route from also being reachable on the first existing router as well.

## Profit! ##

- Watch it work

```bash
[root@ose-test-master-01 ~]# curl http://v3simplebottle-user1.test.os.example.com:49999
<h1> hello OpenShift ninjas</h1>

[root@ose-test-master-01 ~]#  curl -k https://v3simplebottle-user1.test.os.example.com:50000
<h1> hello OpenShift ninjas</h1>
```

# Bugs / Issues #

The following things `oadm router` does not do, and they feel like bugs:

- properly set env `ROUTER_SERVICE_HTTPS_PORT`
- properly set env `ROUTER_SERVICE_HTTP_PORT`
- label the generated `ha-router-teradici-certs` secret as it labels services, dc, and seemingly everything else it created

Apparently this isn't exactly a bug.

- [https://bugzilla.redhat.com/show_bug.cgi?id=1420543](https://bugzilla.redhat.com/show_bug.cgi?id=1420543)

# See Also #

- https://docs.openshift.com/container-platform/3.3/dev_guide/getting_traffic_into_cluster.html
- https://docs.openshift.com/container-platform/3.3/install_config/router/default_haproxy_router.html#using-router-shards
- https://readme.fr/openshift-3-create-my-custom-routerhaproxy/
