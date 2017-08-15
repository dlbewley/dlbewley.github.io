---
title: OpenStack Network Diagram
layout: post
tags:
 - OpenStack
---

What does the networking for OpenStack look like? Maybe something like this.

   | Network            | VLAN  | IP CIDR
---|--------------------|-------|-----------------
N1 | Provisioning (PXE) | V:310 | 172.23.32.0/20
N2 | Internal API | V:311 | 172.23.21.0/24
N3 | Storage Network (Front) | V:312 | 172.23.22.0/24
N4 | Storage Mgmt (Back) | V:313 | 172.23.23.0/24
N5 | External Floating IPs | V:179 | 192.0.179.0/24
N6 | Public API | V:177 | 192.0.177.0/24
N7 | Overcloud Provisioning (Tenant PXE) | V:314 | 172.23.48.0/20
N8 | Provider Network (Tenant VM with physical router) | V:175 | 192.0.175.0/24
N9 | Tenant Network (tunnels) | V:317 | 172.23.96.0/20
N10| IPMI (iDRAC) | V:315 | 172.23.64.0/20
N11| Tenant IPMI (iDRAC) | V:316 | 172.23.80.0/20


[![OpenStack Network Diagram](/images/thumb/openstack-network-pub.png)](/images/openstack-network-pub.png)


