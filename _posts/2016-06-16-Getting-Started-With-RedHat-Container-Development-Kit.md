---
title: Getting Started With RedHat Container Development Kit
layout: post
tags:
 - kubernetes
 - openshift
 - OSE3.1
 - CDK2.0
---

# Register as a RedHat Developer #

- [Obtain a RH login](http://developers.redhat.com/)

- Place credentials in `~/.vagrant.d/Vagrantfile` to enable updates for VMs by automatically registering with RedHat Subscription Manager

```ruby
Vagrant.configure('2') do |config|
 config.registration.username = '<your Red Hat username>'
 config.registration.password = '<your Red Hat password>'
end
```

# Mac OS X Prereqs #

- Install pre-reqs:
    - [Vagrant](https://www.vagrantup.com/)
    - [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

- Download the Vagrant Box [from here](http://developers.redhat.com/products/cdk/get-started/) to `~/Downloads`

# RedHat Container Developer Kit Setup #

- Download Red Hat Container Developer Kit `cdk-2.0.0.zip` [from here](http://developers.redhat.com/downloads/) and unzip to `~/cdk`.

- Follow the [Install Docs](https://access.redhat.com/documentation/en/red-hat-enterprise-linux-atomic-host/version-7/container-development-kit-installation-guide/) to:

    - Install vagrant plugins

    ```bash
    cd ~/cdk/plugins
    vagrant plugin install *.gem
    ```

    - Import the vagrant box to `~/.vagrant.d/boxes/`

    ```bash
    vagrant box add --name cdkv2 ~/Downloads/rhel-cdk-kubernetes-7.2*.x86_64.vagrant-virtualbox.box
    ```

    - Remove `~/Downloads/rhel-cdk-kubernetes-7.2*.x86_64.vagrant-virtualbox.box`

- Start CDK and print helpful environment info

```bash
cd ~/cdk/components/rhel/rhel-ose/
vagrant up
vagrant provision
```

# Getting Started #

- Access the console at https://10.1.2.2:8443/console *User:* _openshift-dev_ *Pass:* _devel_ or User: _admin_ Pass: _admin_

- If you forget where to find the console vagrant can remind you

```bash
cd ~/cdk/components/rhel/rhel-ose/
vagrant service-manager env openshift
```

- Curl the sample app that ships in the CDK

```bash
curl http://helloflask-sample-project.rhel-cdk.10.1.2.2.xip.io/api
```

- SSH to your openshift VM and list the openshift images

```bash
cd ~/cdk/components/rhel/rhel-ose/
vagrant ssh
[vagrant@rhel-cdk ~]$ docker search registry.access.redhat.com/openshift3
INDEX        NAME                                                              DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
redhat.com   registry.access.redhat.com/openshift3/image-inspector             Image Inspector can extract the RPM compos...   0
redhat.com   registry.access.redhat.com/openshift3/jenkins-1-rhel7             Jenkins image which can be used to set up ...   0
redhat.com   registry.access.redhat.com/openshift3/logging-auth-proxy          Container used to enable authorization and...   0
...
```

- Try each of these

```bash
$ docker search registry.access.redhat.com/rhscl
$ docker search registry.access.redhat.com/openshift3
$ docker search registry.access.redhat.com/rhel
$ docker search registry.access.redhat.com/jboss
```

Neat hu?
