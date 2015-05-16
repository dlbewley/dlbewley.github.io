# Ansible is Awesome! Ansible is a Mess! #

So you found [Ansible](http://www.ansible.com), and you were all Woah! Ansible is awesome! Ansibilize all the things! Then you created a git repo and started hacking.

Playbooks look in the current directory to find roles, libraries, and inventories, so naturally you put everything in one big git repo, right?

You tried to follow the [best practices](http://docs.ansible.com/playbooks_best_practices.html) for writing playbooks, you created [roles](http://docs.ansible.com/playbooks_roles.html), and maybe you wrote a [filter plugin](http://docs.ansible.com/developing_plugins.html) or a [custom module](http://docs.ansible.com/developing_modules.html) for configuring an application unique to your environment. Eventually you wound up with a big repo of playbooks and inventory files and buried roles and realized there must be a better way to do this.

# There Are No Masters #

Ansible does not use an agent on the managed host, and it doesn't have a central server (unless you use [Ansible Tower](http://docs.ansible.com/tower.html), kinda). Basically anyone on your team can install ansible on their workstation and start adding to your mess or create another one just making things worse and worse.

**How do you break this down into managable pieces?**

- How to you separate your playbooks and your roles?
- How do you properly create roles?
- How do you access roles created by others on your team or on [Ansible Galaxy](http://galaxy.ansible.com/)?
- use your own roles along with other people's roles in the same playbook?
- make sure all the above are available on demand by other admins?

# Playbook Repos Provide a Context #

Most of my use of Ansible is for provisioning of services rather than deploying a specific application. So, my ansible bits are not inside of an application's git repo. How should they be tracked then?

How about creating a repo called `playbook-<task>`, so for provisioning something large like a multi-server Zimbra install, I create a repo called `playbook-zimbra`. The playbook repo will of course have at least one playbook yaml file, but it also defines what I call an ansible runtime context: config file, inventory file, group variables, host variables, roles, libraries, etc.

When creating a role within a playbook, the goal should be to make it as general as possible. Once a role is able to define its own reasonable defaults and parameterize all options as variables, it should be [split out](http://guifreelife.com/blog/2015/03/15/Split-Ansible-Git-Repo-and-Retain-Commit-History/) of the playbook repo into its own repo.

# Creating Roles #

It is usually better to follow an existing convention, even when it may be overkill, because it makes it easier for others and future-you to figure out what's going on. _Never underestimate the value of standards and conventions!_ Sometimes coming to a convention isn't easy though. 

Ansible does make it easy to create a skeleton for your role that is consistent with the rest of the world. 

Just start by running `ansible-galaxy init role_name`. Most likely you want to create a git repo out of that role, so do that too.

{% highlight %}
cd ~/src
ansible-galaxy init foorole
cd foorole
git init
git add .
git commit -m firstsies
git remote add origin git@github.com:dlbewley/foorole.git
git push -u origin master
{% endhighlight %}

Now you have a git repo named _foorole_. You might rename the repo to _ansible-foorole_, but I prefer to push the repo with the name of _foorole_ and only name my working directory _ansible-foorole_.

{% highlight %}
# create a foorole repo on github then push to it
git remote add origin git@github.com:dlbewley/foorole.git
git push -u origin master
cd ~/src
git clone git@github.com:dlbewley/foorole.git ansible-foorole
{% endhighlight %}

# Ansible Galaxy #

[Ansible Galaxy](http://galaxy.ansible.com/) is essentially a framework that makes it super simple to use a stranger's role in your playbook. Roles are refrenced as `stranger_name.role_name`.

**You can reference these strangers' roles in a few ways.**

- [Install](http://docs.ansible.com/galaxy.html#installing-roles) them individually:
 `ansible-galaxy install username.rolename`

- List them one per line in a `requirements.txt` and install all at once:
 `ansible-galaxy install -r requirements.txt`

- List them as a [role dependency](http://docs.ansible.com/playbooks_roles.html#role-dependencies) in another role to be install automatically:
  Add to `dependencies: []` in `foorole/meta/main.yml`

Role dependencies [currently](https://github.com/ansible/ansible/blob/devel/bin/ansible-galaxy) only work with roles hosted on [Ansible Galaxy](http://galaxy.ansible.com). See `fetch_role()`, basically it assumes it will be downloading a `.tar.gz` of a git repo, most likely from github.

{% highlight python %}
    # first grab the file and save it to a temp location
    if '://' in role_name:
        archive_url = role_name
    else: 
        archive_url = 'https://github.com/%s/%s/archive/%s.tar.gz' % (role_data["github_user"], role_data["github_repo"], target)
    print "- downloading role from %s" % archive_url
{% endhighlight %}

- List them in `requirements.yml`. More on this below.

**But maybe you don't trust these strangers. Then what?**

# Build Your Own Galaxy # 

In addition to `requirements.txt` which assumes Ansible Galaxy as the source of the role, as of v1.8 there is support for a `requirements.yml` which let's you point at a `.tar.gz` or SCM repo somewhere else, like your own local [gitlab](https://about.gitlab.com/).

# Installing Roles #

There are tools or at least a tool, called [Ansible Role Manager](http://mirskytech.github.io/ansible-role-manager/), which can help you manage roles, but another option is just a Makefile.

Something like this will allow you to type `make install` to resolve your `requirements.yml`:

{% highlight make %}
.PHONY: galaxy-install ping

install: galaxy-install

galaxy-install:
	ansible-galaxy install -r requirements.yml --force

ping:
	ansible all -i hosts -m ping
{% endhighlight %}

Where do these roles get installed? Ansible-galaxy will install these roles to the first directory found in your [roles_path](http://docs.ansible.com/intro_configuration.html#roles-path). Remember that. We can take advantage of that.

## Roles Path ##

If you are working on a playbook which may have roles stored along side it, and you want to use roles from Ansible Galaxy, what do you do?

Remember, I just said that that `ansible-galaxy install` places roles in the first directory in your `roles_path`? Just create a ansible.cfg in your playbook directory that looks something like this:

{% highlight ini %}
[defaults]
remote_user = root
inventory_file = hosts
roles_path = required-roles:roles
{% endhighlight %}

Then add `required-roles` to your `.gitignore`. Now, you can disentangle your roles and external roles within your playbook. I have to give credit to [@command_tab](https://twitter.com/command_tab) for changing my life with this tip. :)

# Organizing Repos When Working with Other Admins #

You might run ansible from a laptop, your workstation, a VM. Your colleagues may do the same. How do you make sure each environment is consistent and compatible?

As you scale up use of Ansible on your team, what sort of groundwork can you lay for keeping things organized? 
I suggest a `Ansible` name space in a Gitlab instance then try to put all the ansible related bits in there as long as it makes sense. Sometimes a playbook is tightly bound to an application and it seems to make sense to keep it in a `ansible` sub dir of that repo.

How about some repo naming conventions? 

- [Role](http://docs.ansible.com/playbooks_roles.html) repos should be named after the role and have no prefix
- [Module](http://docs.ansible.com/developing_modules.html) repos should have a prefix of `module-`
- [Plugin](http://docs.ansible.com/developing_plugins.html) repos should have a prefix of `plugin-`
- [Playbook](http://docs.ansible.com/playbooks.html) repos should have a prefix of `playbook-`

# Still To Be Determined #

Distributing [inventory](http://docs.ansible.com/intro_inventory.html) is a problem I haven't quite figured out yet. For now, each playbook has it's own static inventory file and that works just fine for somethings, but it isn't generally scalable.

A [dynamic inventory](http://docs.ansible.com/intro_dynamic_inventory.html) script which queries a LDAP directory seems like the obvious choice, but with thousands of hosts how can this be done efficiently? [Patterns](http://docs.ansible.com/intro_patterns.html) are applied to host lists after the inventory is constructed. Somehow, the limitation needs to be included in the LDAP filter.
