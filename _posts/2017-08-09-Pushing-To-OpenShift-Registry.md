---
title: How to push an image to an unexposed OpenShift Docker registry
layout: post
tags:
 - openshift
 - docker
---

How do I push an image to the OpenShift Docker registry if it is not exposed outside the cluster?

**Login to a member node**

Get on a machine that has docker and participates in the cluster SDN or can somehow access that network. (eg. `172.30.0.0/16`)

**Get the IP of the registry**

{% highlight bash %}{% raw %}
oc get svc docker-registry -n default --template "{{ .spec.clusterIP }}"
SVC_REGISTRY=$(oc get svc docker-registry -n default --template "{{ .spec.clusterIP }}")
{% endraw %}{% endhighlight %}

**Get a token for your session**

```bash
oc whoami -t
r4bjk9MrFcWz8SjR688zvbAbwnI1yoqNbtdhN1sMaeo
TOKEN=$(oc whoami -t)
```

**Login to the docker registry**

```bash
docker login -u $(oc whoami) -p $TOKEN  $SVC_REGISTRY:5000
```

**Commit the container and / or tag the image**

```bash
docker commit <container_id> $SVC_REGISTRY:5000/project/image:tag
```

**Push the tagged image**

```bash
docker push $SVC_REGISTRY:5000/project/image:tag
```

## See Also ##

- [OpenShift Managing Images](https://docs.openshift.com/container-platform/latest/dev_guide/managing_images.html)
