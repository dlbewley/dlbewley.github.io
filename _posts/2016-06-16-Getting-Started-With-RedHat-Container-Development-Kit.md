---
title: Getting Started With RedHat Container Development Kit
layout: post
tags:
 - kubernetes
 - openshift
 - OSE3.1
 - OSE3.2
 - CDK2.0
 - CDK2.1
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

# Getting Started #

- Start CDK and print helpful environment info

```bash
cd ~/cdk/components/rhel/rhel-ose/
vagrant up
vagrant provision
```

- Access the console at [https://10.1.2.2:8443/console](https://10.1.2.2:8443/console/) using a credential below:
  - *User:* _openshift-dev_ *Pass:* _devel_
  - *User:* _admin_ *Pass:* _admin_

- If you forget where to find the console vagrant can remind you

```bash
cd ~/cdk/components/rhel/rhel-ose/
vagrant service-manager env openshift
```

- Curl the sample app that ships in the CDK

```bash
curl http://helloflask-sample-project.rhel-cdk.10.1.2.2.xip.io/api
```

How the heck did that work?! Checkout [xip.io](http://xip.io).

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

# Updating CDK #

CDK 2.1 is out now with OpenShift Enterprise 3.2. Let's update!


- Download Red Hat Container Developer Kit `cdk-2.1.0.zip` [from here](http://developers.redhat.com/downloads/) and unzip to `~/cdk-2.1`.

```bash
mv ~/cdk ~/cdk-2.0
ln -s ~/cdk-2.1 ~/cdk
```

- Update Vagrant Plugins

```bash
$ cd ~/cdk/plugins

$ vagrant plugin list
landrush (0.15.3)
vagrant-dnsmasq (0.1.1)
vagrant-hostmanager (1.6.0)
vagrant-registration (1.2.1)
  - Version Constraint: 1.2.1
vagrant-service-manager (1.0.1)
  - Version Constraint: 1.0.1
vagrant-share (1.1.4, system)
vagrant-sshfs (1.1.0)
  - Version Constraint: 1.1.0

$ ls *gem
vagrant-registration-1.2.2.gem          vagrant-service-manager-1.1.0.gem       vagrant-sshfs-1.1.0.gem

$ vagrant plugin install vagrant-registration-1.2.2.gem
Installing the 'vagrant-registration-1.2.2.gem' plugin. This can take a few minutes...
Installed the plugin 'vagrant-registration (1.2.2)'!
```

- Download the latest [Red Hat Enterprise Linux 7.2 Vagrant box for VirtualBox](http://developers.redhat.com/products/cdk/get-started/)

- Update the vagrant box **This will replace your current environment, so modify if you with to keep the 2.0 environment around!**

```bash
$ vagrant box add --force --name cdkv2 rhel-cdk-kubernetes-7.2*.x86_64.vagrant-virtualbox.box
```

Now go back to [Getting Started](#getting-started) and fire up your new CDK 2.1 VM.
