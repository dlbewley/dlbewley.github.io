---
title: Ansible CMDB Inventory and Facts Reporting
layout: post
tags:
 - ansible
 - automation
---

You just deployed a complex multi-host app using Ansible. Wouldn't it be helpful to see a overview of the deployment including hardware details?

I just found [ansible-cmdb](https://github.com/fboender/ansible-cmdb) which combines info from the Ansible inventory and discovered facts to create a detailed HTML report akin to a Configuration Management Database.

To use it in your playbook dir, just create a directory to hold facts discovered by the [setup module](http://docs.ansible.com/ansible/setup_module.html) then generate the report.

{% highlight bash %}
mkdir cmdb
ansible all -i hosts -m setup --tree cmdb/
ansible-cmdb -i hosts cmdb/ > cmdb.html
{% endhighlight %}

Now go take a look at `cmdb.html`. Here is an example report.

![Screenshot Overview](https://raw.githubusercontent.com/fboender/ansible-cmdb/master/contrib/screenshot-overview.png)
