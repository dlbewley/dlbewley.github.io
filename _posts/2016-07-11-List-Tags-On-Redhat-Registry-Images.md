---
title: How to List Tags On Redhat Registry Images
layout: post
tags:
 - Docker
 - openshift
 - OSE3.2
---


Ever gone to [RedHat's container registry](https://access.redhat.com/search/#/container-images) to search for an image and been left wondering what versions exist? Ever been frustrated by the inconsistent tag format? Is there a _v_ or is there not a _v_? Me too.

Docker Hub has progressed to [v2](https://docs.docker.com/registry/spec/api/), while the RedHat registry is still [v1](https://docs.docker.com/v1.6/reference/api/registry_api/).

You can use curl to query the registry API and [list the tags](https://docs.docker.com/v1.6/reference/api/registry_api/#list-repository-tags) like this:

```bash
$ curl -s https://registry.access.redhat.com/v1/repositories/openshift3/${image}/tags | jq .
```

**Example:**

```json
[root@ose-prod-master-01 ~]# curl -s https://registry.access.redhat.com/v1/repositories/openshift3/logging-kibana/tags | jq .
{
  "3.1.0": "2c5acfd9c89c5b122e41b7ca13c6847aa9183bd41076ac9a3836a28da5b82bb4",
  "3.1.0-1": "10b5df01eb1793076b746f27c5b439bde2d73bdf988d805fac450255c22d904b",
  "3.1.0-4": "2c5acfd9c89c5b122e41b7ca13c6847aa9183bd41076ac9a3836a28da5b82bb4",
  "3.1.1": "d15c32b3f8e985bea5e2401989b33413f7e30627bd1c9e108c7a6a5e0b0b0fc4",
  "3.1.1-5": "1d7701631584635d9f07e62f2ba0dd6fea0a2eb2ece02010b2630b2e9aaf9af7",
  "3.1.1-6": "5a922f7be584559218fa5e70e9ee159650b31ef0ac68d9e8b3a9ca7d45ef491c",
  "3.1.1-7": "3ce38d90561782969f98f387ba96b0e8411ee58a76878fcbd194ed64f54150a9",
  "3.1.1-8": "d15c32b3f8e985bea5e2401989b33413f7e30627bd1c9e108c7a6a5e0b0b0fc4",
  "3.2.0": "38480621ccf59391f18db56caa7a83ad5ab262d648548ee60b6d993936366ea8",
  "3.2.0-3": "51ec0188aa11dc11c6f24a4bf69d8b95a8a996209d3bd17fa40bdb65e20f8b89",
  "3.2.0-4": "38480621ccf59391f18db56caa7a83ad5ab262d648548ee60b6d993936366ea8",
  "3.2.1": "7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef",
  "3.2.1-3": "7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef",
  "latest": "7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef",
  "v3.1": "d15c32b3f8e985bea5e2401989b33413f7e30627bd1c9e108c7a6a5e0b0b0fc4",
  "v3.2": "7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef"
}
```
