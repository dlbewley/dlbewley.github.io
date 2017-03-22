---
title: How to List Tags On Redhat Registry Images
layout: post
tags:
 - docker
 - openshift
 - OSE3.2
---


Ever gone to [RedHat's container registry](https://access.redhat.com/search/#/container-images) to search for an image and been left wondering what versions exist? Ever been frustrated by the inconsistent tag format? Is there a _v_ or is there not a _v_? Me too.

Docker Hub has progressed to [v2](https://docs.docker.com/registry/spec/api/), while the RedHat registry is still [v1](https://docs.docker.com/v1.6/reference/api/registry_api/) at the moment. As long as you use the right syntax, you can use curl to query the registry API and [list the tags](https://docs.docker.com/v1.6/reference/api/registry_api/#list-repository-tags) like this:

```bash
# v1 registry
$ curl -s https://registry.access.redhat.com/v1/repositories/openshift3/${image}/tags | jq .
# v2 registry
$ curl -sL https://registry.access.redhat.com/v2/openshift3/${image}/tags/list
```

**Example:**

```bash
$ curl -s https://registry.access.redhat.com/v1/repositories/openshift3/logging-kibana/tags | jq .
```
```json
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

Note that Docker images are content addressable, so the tags `3.2.1-3`, `3.2.1`, `v3.2`, and `latest` above all point to the same image _7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef_.

# Update September 2016 #

I just happened to be working on [RHSA-2016:1836](https://access.redhat.com/errata/RHSA-2016:1836) Kibana security advisory which recommends updating to `openshift3/logging-kibana:3.2.1-5` image.

Take a look at the tags as of 2016-09-18. Notice that `latest` is still _7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef_ like it was back in July, but `3.2.1` and `v3.2` have moved to point to `3.2.1-5` _982e831f5fba03abca20e1f8fbbc8d31ddf8747c5ee8170ddfcba93018e5730d_. I would expect `latest` to match `3.2.1` right now.

```bash
$ curl -s https://registry.access.redhat.com/v1/repositories/openshift3/logging-kibana/tags | jq .
```
```json
{
  "3.1.0": "2c5acfd9c89c5b122e41b7ca13c6847aa9183bd41076ac9a3836a28da5b82bb4",
  "3.1.0-1": "10b5df01eb1793076b746f27c5b439bde2d73bdf988d805fac450255c22d904b",
  "3.1.0-4": "2c5acfd9c89c5b122e41b7ca13c6847aa9183bd41076ac9a3836a28da5b82bb4",
  "3.1.1": "83a4455f3d606c41824d834df224e0579f074d9dec75a6fcc148dac35da13b1b",
  "3.1.1-10": "83a4455f3d606c41824d834df224e0579f074d9dec75a6fcc148dac35da13b1b",
  "3.1.1-5": "1d7701631584635d9f07e62f2ba0dd6fea0a2eb2ece02010b2630b2e9aaf9af7",
  "3.1.1-6": "5a922f7be584559218fa5e70e9ee159650b31ef0ac68d9e8b3a9ca7d45ef491c",
  "3.1.1-7": "3ce38d90561782969f98f387ba96b0e8411ee58a76878fcbd194ed64f54150a9",
  "3.1.1-8": "d15c32b3f8e985bea5e2401989b33413f7e30627bd1c9e108c7a6a5e0b0b0fc4",
  "3.2.0": "38480621ccf59391f18db56caa7a83ad5ab262d648548ee60b6d993936366ea8",
  "3.2.0-3": "51ec0188aa11dc11c6f24a4bf69d8b95a8a996209d3bd17fa40bdb65e20f8b89",
  "3.2.0-4": "38480621ccf59391f18db56caa7a83ad5ab262d648548ee60b6d993936366ea8",
  "3.2.1": "982e831f5fba03abca20e1f8fbbc8d31ddf8747c5ee8170ddfcba93018e5730d",
  "3.2.1-3": "7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef",
  "3.2.1-5": "982e831f5fba03abca20e1f8fbbc8d31ddf8747c5ee8170ddfcba93018e5730d",
  "latest": "7ac03c5eb56e15575c86d57ced2387c844d17469f4fda7c7928a660b77bf3aef",
  "v3.1": "83a4455f3d606c41824d834df224e0579f074d9dec75a6fcc148dac35da13b1b",
  "v3.2": "982e831f5fba03abca20e1f8fbbc8d31ddf8747c5ee8170ddfcba93018e5730d"
}
```

It seems like Red Hat is still coming to terms with release management in this new image based world. There isn't even information in the eratta on how to update to new images, instead one is referred to the info on [how to apply RPM updates](https://access.redhat.com/articles/11258). Inconsistent formatting of tags have caused [other problems](https://bugzilla.redhat.com/show_bug.cgi?id=1339754) for updates. Better them than me, though.
