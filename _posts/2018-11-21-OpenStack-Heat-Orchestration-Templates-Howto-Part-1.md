---
title: How to Create and Use OpenStack Heat Orchestration Templates Part 1
layout: post
tags:
 - heat
 - openstack
---

OpenStack enables automated creation of resources such as networks, routers, and servers using [Heat](http://docs.openstack.org/developer/heat/template_guide/hot_spec.html) Orchestration Templates. If you are new to OpenStack and are using a [TripleO](https://docs.openstack.org/tripleo-docs/latest/) based distribution you may have seen them up close and personal without knowing it. What follows is a very basic exploration of Heat.

Heat templates are written in YAML format, and you can quickly see from the documentation that a basic template will likely have 4 sections:

{% highlight yaml %}{% raw %}
# templates will use features as of this release of OpenStack
heat_template_version: queens
  ...
# variables to inform the creation of resources to follow 
parameters:
  ...
# the things the template will be creating
resources:
  ...
# information about the things that were created
outputs:
  ...
{% endraw %}{% endhighlight %}

The [heat_template_version](https://docs.openstack.org/heat/latest/template_guide/hot_spec.html#heat-template-version) specifies the features expected to be supported by the template to follow. You may see dated versions, but since Newton the release name is supported. I would use that if the template is not going to be used on a cloud older than that.

The [parameters](https://docs.openstack.org/heat/latest/template_guide/hot_spec.html#parameters-section) section identifies any variables that will be used later in the template. This is where a place holder for a server name, or the flavor to use would be defined. Parameters can have a default value and miscellaneous constraints applied for validation.

Objects like a server instance are known as [resources](https://docs.openstack.org/heat/latest/template_guide/hot_spec.html#resources-section). Anything that the template will create or instantiate will be listed in the `resources` section and will be informed by the `parameters` section.

Once a template has been "launched" or fed to OpenStack it will create what is called a `stack` of all the resources requested. There are likely some obvious questions you want to ask about a stack like: _"What is the name of the server I just created?"_ or _"What is the IP of my server?"_. The `outputs` section makes it easy to expose that information.

# Simple Example Template

Here is a template that will create a server and attach it to a network, but it assumes that a network exists. In my case I already have a network named `provider192`.

{% highlight yaml %}{% raw %}
# file: example-1.yaml
# i never plan to run on OpenStack < Queens
heat_template_version: queens

description: Create a server instance

parameters:
  # define a variable named "image" to use later
  image:
    description: Name of image from Glance to create an instance from.
    type: string
    # Do you have an image named this already?
    default: cirros-0.4.0.raw
  flavor:
    description: Instance flavor to build.
    type: string
    # Do you have a flavor named this already?
    default: m1.small
    constraints:
      # only accept valid flavors for this parameter
      - custom_constraint: nova.flavor

resources:
  # create a resource named "my_instance"
  my_instance:
    # the my_instance resource is a server
    type: OS::Nova::Server
    properties:
      # use the image parameter / variable defined above
      image: { get_param: image }
      flavor: { get_param: flavor }

outputs:
  name:
    description: The name of the server created in this stack.
    value: { get_attr: [ my_instance, name ] }
{% endraw %}{% endhighlight %}

## Create Example-1 Stack with Defaults

Save that to a file called `example-1.yaml`. Edit the default flavor and image values to match what exists in your cloud.

Now create a stack from the template. The command looks like this:

```bash
$ openstack stack create -t <template_filename.yaml> <resultant_stack_name>
```

```bash
$ openstack stack create -t example-1.yaml example-1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| id                  | 218fedd6-efdd-4c2b-ac82-7f15bf1c4235 |
| stack_name          | example-1                            |
| description         | Create a server instance             |
| creation_time       | 2018-11-22T00:21:16Z                 |
| updated_time        | None                                 |
| stack_status        | CREATE_IN_PROGRESS                   |
| stack_status_reason | Stack CREATE started                 |
+---------------------+--------------------------------------+
```

Wait a minute or two, and then list the stack and see what outputs it can show us.

```bash
$ openstack stack list
+--------------------------------------+------------+-----------------+----------------------+--------------+
| ID                                   | Stack Name | Stack Status    | Creation Time        | Updated Time |
+--------------------------------------+------------+-----------------+----------------------+--------------+
| 218fedd6-efdd-4c2b-ac82-7f15bf1c4235 | example-1  | CREATE_COMPLETE | 2018-11-22T00:21:16Z | None         |
+--------------------------------------+------------+-----------------+----------------------+--------------+

$ openstack stack output list example-1
+------------+-----------------------------------------------+
| output_key | description                                   |
+------------+-----------------------------------------------+
| name       | The name of the server created in this stack. |
+------------+-----------------------------------------------+
```

That looks familiar, what does it contain?

```bash
$ openstack stack output show example-1 name
+--------------+-----------------------------------------------+
| Field        | Value                                         |
+--------------+-----------------------------------------------+
| description  | The name of the server created in this stack. |
| output_key   | name                                          |
| output_value | example-1-my_instance-omfgcgk3w7nd            |
+--------------+-----------------------------------------------+
```

There is the name of the server we just created! How about something more to the point for use in a script.

```bash
$ openstack stack output show example-1 name -f value -c output_value
example-1-my_instance-omfgcgk3w7nd 
```

Take a look at the whole stack.

```bash
$ openstack stack show example-1
+-----------------------+-------------------------------------------------------------------------------------------------------------------------------------+
| Field                 | Value                                                                                                                               |
+-----------------------+-------------------------------------------------------------------------------------------------------------------------------------+
| id                    | 218fedd6-efdd-4c2b-ac82-7f15bf1c4235                                                                                                |
| stack_name            | example-1                                                                                                                           |
| description           | Create a server instance                                                                                                            |
| creation_time         | 2018-11-22T00:21:16Z                                                                                                                |
| updated_time          | None                                                                                                                                |
| stack_status          | CREATE_COMPLETE                                                                                                                     |
| stack_status_reason   | Stack CREATE completed successfully                                                                                                 |
| parameters            | OS::project_id: 1686c505946841259bfda1f1c206456d                                                                                    |
|                       | OS::stack_id: 218fedd6-efdd-4c2b-ac82-7f15bf1c4235                                                                                  |
|                       | OS::stack_name: example-1                                                                                                           |
|                       | flavor: m1.small                                                                                                                    |
|                       | image: cirros-0.4.0.raw                                                                                                             |
|                       |                                                                                                                                     |
| outputs               | - description: The name of the server created in this stack.                                                                        |
|                       |   output_key: name                                                                                                                  |
|                       |   output_value: example-1-my_instance-omfgcgk3w7nd                                                                                  |
|                       |                                                                                                                                     |
| links                 | - href: https://openstack.example.com:13004/v1/1686c505946841259bfda1f1c206456d/stacks/example-1/218fedd6-efdd-4c2b-ac82-7f15bf1c4235|
|                       |   rel: self                                                                                                                         |
|                       |                                                                                                                                     |
| parent                | None                                                                                                                                |
| disable_rollback      | True                                                                                                                                |
| deletion_time         | None                                                                                                                                |
| stack_user_project_id | 4b22b80f4a1843f7a90b13e54375b4d5                                                                                                    |
| capabilities          | []                                                                                                                                  |
| notification_topics   | []                                                                                                                                  |
| tags                  | None                                                                                                                                |
| timeout_mins          | None                                                                                                                                |
| stack_owner           | None                                                                                                                                |
+-----------------------+-------------------------------------------------------------------------------------------------------------------------------------+
```

And now at the server it created. Notice that it attached to a network I had already created, named _provider192_.

```bash
$ openstack server show example-1-my_instance-omfgcgk3w7nd
+-----------------------------+----------------------------------------------------------+
| Field                       | Value                                                    |
+-----------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig           | MANUAL                                                   |
| OS-EXT-AZ:availability_zone | nova                                                     |
| OS-EXT-STS:power_state      | Running                                                  |
| OS-EXT-STS:task_state       | None                                                     |
| OS-EXT-STS:vm_state         | active                                                   |
| OS-SRV-USG:launched_at      | 2018-11-22T00:21:31.000000                               |
| OS-SRV-USG:terminated_at    | None                                                     |
| accessIPv4                  |                                                          |
| accessIPv6                  |                                                          |
| addresses                   | provider192=192.0.2.14                                   |
| config_drive                |                                                          |
| created                     | 2018-11-22T00:21:22Z                                     |
| flavor                      | m1.small (af20508a-5986-4bf3-9e9f-33ca598e1563)          |
| hostId                      | f1e7461e138a94335e579cb81cd95933fc3879b263a3109281d203ba |
| id                          | ce6e7e7a-f091-4fe8-bde0-b0ae3d3664d7                     |
| image                       | cirros-0.4.0.raw (a034ee47-4642-4d21-9b08-11774808a822)  |
| key_name                    | None                                                     |
| name                        | example-1-my_instance-omfgcgk3w7nd                       |
| progress                    | 0                                                        |
| project_id                  | 1686c505946841259bfda1f1c206456d                         |
| properties                  |                                                          |
| security_groups             | name='default'                                           |
| status                      | ACTIVE                                                   |
| updated                     | 2018-11-22T00:21:31Z                                     |
| user_id                     | e2041d2848be40f6b6fb86709b260ae1                         |
| volumes_attached            |                                                          |
+-----------------------------+----------------------------------------------------------+
```

Throw away this stack. This will delete the stack and any resources it created, such as the `example-1-my_instance-omfgcgk3w7nd` server.

```bash
$ openstack stack delete example-1 --yes
```

## Create Example-1 Stack with Supplied Parameters

Let's see how to pass in parameter values and demonstrate the effect of the [custom constraint](https://docs.openstack.org/heat/latest/template_guide/hot_spec.html#custom-constraint) we placed on the flavor parameter.

We know there is a parameter named `flavor` we can provide a value to override the default value when we create a stack from our template.

```bash
# flawed example
$ openstack stack create -t example-1.yaml --parameter flavor=nonexistant example-fail
ERROR: Parameter 'flavor' is invalid: Error validating value 'nonexistant': No Flavor matching {'name': u'nonexistant'}. (HTTP 404)
```

Since the flavor was not found, our stack was not created. In truth, without that custom constraint a similar error would have been produced.

```bash
# flawed example with custom constraint removed
$ openstack stack create -t example-1.yaml --parameter flavor=nonexistant example-fail
ERROR: Property error: : resources.my_instance.properties.flavor: : Error validating value 'nonexistant': No Flavor matching {'name': u'nonexistant'}. (HTTP 404)
```

Of course if you pass the name of a flavor that exists in your cloud there will be no error.

```bash
# example with custom constraint removed
$ openstack stack create -t example-1.yaml --parameter flavor=m1.large example-1
```


## Create Example-1 Stack with Environment File

Instead of passing the parameters on the command line you can also use what is called an environment file. An environment file should include a `parameter_defaults` section containing the parameters it will define and the values to supply. For example, this will provide a new value for the `flavor` parameter when coupled with our `example-1.yaml` template.

{% highlight yaml %}{% raw %}
# file: environment-1.yaml

parameter_defaults:
  flavor: m1.medium
{% endraw %}{% endhighlight %}

Make sure the stack is gone and then recreate it with the above environment file.

```bash
$ openstack stack delete example-1 --yes
$ openstack stack create -t example-1.yaml -e environment-1.yaml example-1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| id                  | 06fbdce6-5ac5-49b0-be54-7766fd1afdef |
| stack_name          | example-1                            |
| description         | Create a server instance             |
| creation_time       | 2018-11-22T01:48:03Z                 |
| updated_time        | None                                 |
| stack_status        | CREATE_IN_PROGRESS                   |
| stack_status_reason | Stack CREATE started                 |
+---------------------+--------------------------------------+
```

See what resources the stack created.

```bash
$ openstack stack resource list example-1
+---------------+--------------------------------------+------------------+-----------------+----------------------+
| resource_name | physical_resource_id                 | resource_type    | resource_status | updated_time         |
+---------------+--------------------------------------+------------------+-----------------+----------------------+
| my_instance   | 90794a87-7db8-4220-a1c9-6f2e4b68429b | OS::Nova::Server | CREATE_COMPLETE | 2018-11-22T01:48:04Z |
+---------------+--------------------------------------+------------------+-----------------+----------------------+
```

Remember `my_instance` is the name of the Heat resource not the Nova server, but the physical_resource_id is the UUID of the server.

What flavor is that server instance? Instead of the default `m1.small`, it should be `m1.medium` per our environment file.


```bash
$ openstack server show 90794a87-7db8-4220-a1c9-6f2e4b68429b -c flavor -c name
+--------+--------------------------------------------------+
| Field  | Value                                            |
+--------+--------------------------------------------------+
| flavor | m1.medium (fa4efbb9-6f43-4a29-aa1a-fa1069681ad6) |
| name   | example-1-my_instance-c3pg4dyb5l6n               |
+--------+--------------------------------------------------+
```

**Bingo!**

I hope this basic example of using Heat Orchestration Templates was informative.
I'll try to follow up with some more comprehensive examples. Check back in a week or two!

# See Also

**Some helpful blogs and documentation**

- Rackspace [OpenStack Orchestration In Depth, Part I: Introduction to Heat](https://developer.rackspace.com/blog/openstack-orchestration-in-depth-part-1-introduction-to-heat/)
- [OpenStack Resource Types](http://docs.openstack.org/developer/heat/template_guide/openstack.html)
- [OpenStack User Guide Create and Manage Stacks](https://docs.openstack.org/newton/user-guide/cli-create-and-manage-stacks.html)
- [HOT Template Specification](http://docs.openstack.org/developer/heat/template_guide/hot_spec.html)
- [OpenStack Resource Types](https://docs.openstack.org/heat/latest/template_guide/openstack.html) - Go here when looking up resource parameters and outputs.
- [TripleO Documentation](https://docs.openstack.org/tripleo-docs/latest/)
- [Heat Template Composition](https://docs.openstack.org/heat/latest/template_guide/composition.html)
- Stratoscale [Best Practices for OpenStack Heat Templates](https://www.stratoscale.com/blog/openstack/best-practices-openstack-heat-templates/)
- [DevOpsPro 2016](http://devops-pro-2016.armyofevilrobots.com/) is a great tutorial by example!
- [Heat 101](http://yauuu.me/heat-101.html) by gchenuet
