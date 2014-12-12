Ansible is Awesome! Ansible is a Mess! (draft)
===

So you found [Ansible](http://www.ansible.com), and you were like "Woah! Ansible is awesome!". You tried to follow the [best practices](http://docs.ansible.com/playbooks_best_practices.html) for writing playbooks, you created [roles](http://docs.ansible.com/playbooks_roles.html), and maybe your wrote a [filter plugin](http://docs.ansible.com/developing_plugins.html) or a [custom module](http://docs.ansible.com/developing_modules.html) for configuring an application unique to your environment. Then you started to feel like that things may have gotten out of hand and you've created a mess!

OK, maybe I'm just describing me and my environment.

Playbooks, look in the current directory to find roles, libraries, and inventories, so naturally you put everything in one big git repo. How do you manage this? How do you allow others to use it and contribute to it? What do you name all those inventories and playbooks? There must be a way to fix this by tweaking [ansible.cfg](http://docs.ansible.com/intro_configuration.html) or something, right?

There are no masters
===

Ansible does not use an agent on the managed host, and it doesn't have a central server (unless you use [Ansible Tower](http://docs.ansible.com/tower.html), kinda). Basically anyone on your team who can install ansible on their workstation, has credentials on the host to be managed, and has access to your bits can use the big mess you just made. So naturally you put your mess in a big git repo to make it accessible.

But how do you break this down into managable pieces usable from anywhere, and where do you use it from?

**How do you:**
- scalably manage roles?
- keep track of your playbooks?
- make sure all the above available on demand by other admins?

Creating Roles
===
I've found it is usually better to follow convention even when it may be overkill, because it makes it easier for others and future-you to figure out what's going on. _Never underestimate the value of standards and conventions!_ Sometimes coming to a convention isn't easy though. Ansible does make it easy to create a skeleton for your role that is consistent with the rest of the world. 

Just start by running `ansible-galaxy init role_name`. Most likely you want to create a git repo out of that role, so do that too.

{% highlight %}
ansible-galaxy init foorole
cd foorole
git init
git add .
git commit -m firstsies
{% endhighlight %}

TODO how do you rename the dir ansible-role-foo and still reference foo?


Ansible Galaxy
===

[Ansible Galaxy](http://galaxy.ansible.com/) is essentially a framework that makes it super simple to use a stranger's role in your playbook. Roles are refrenced as `stranger_name.role_name`.

You can use these roles in a couple of ways. 

- You can install them manually:
 `ansible-galaxy install username.rolename`
- List them as a dependency in `foorole/meta/main.yml`
- List them one per line in a `requirements.txt`
 `ansible-galaxy install -r requirements.txt`

But maybe you don't trust strangers. Then what?

In addition to `requirements.txt` which assumes [ansible-galaxy](http://galaxy.ansible.com/) as the source of the role, as of v1.8 there is support for a `requirements.yml` which let's you point at a `.tar.gz` somewhere else.

TODO maybe just listing dependencies is a thing too
http://docs.ansible.com/playbooks_roles.html#role-dependencies

Working with Other Admins
===

Now how do you manage your systems?

Multiple Ansible Stations
---
You might run ansible from a laptop, your workstation, a VM. Your colleagues may do the same. How do you make sure each environment is consistent and compatible?

Inventory
---
http://docs.ansible.com/developing_inventory.html
