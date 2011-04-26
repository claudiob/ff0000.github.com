---
layout: post
title: "Spawning Amazon EC2 instances in Python"
author: claudiob
---

At RED Interactive we create at least five new Amazon EC2 instances for website we work on: one for staging, two for the production database and two for the production code. To keep consistency among these instances, we have created a Python command-line tool called [ec2selector](https://github.com/ff0000/ec2selector) that we are now releasing on GitHub as an open source project. <!--more-->

## Why boto is not enough

`ec2selector` expands over the work of [boto](http://boto.cloudhackers.com/), the unrivalled Python package that provides interfaces to Amazon Web Services. `boto` makes it easy to create and run any number of Amazon EC2 instances without ever leaving the Python shell:

{% highlight python %}
from boto.ec2.connection import EC2Connection 
conn = EC2Connection('<aws access key>', '<aws secret key>')
image = conn.get_image('ami-20b65349')
image.run()
{% endhighlight %}
    
For this approach, you need to specify the identifier of the Amazon Machine Image (AMI) that you want the new instances to be based on. AMIs are encrypted machine image stored in Amazon S3 which contain all of the information necessary to boot instances on EC2: operative system, architecture, virtualization type, and so on. In the previous command, 'ami-20b65349' refers to a Fedora Core 4 i386 image.

The drawback is that AMIs do not live forever. As operative systems get upgraded, so do AMIs. And older versions can be deleted from Amazon to leave place to new ones.

## We just want an Ubuntu machine

The solution we have come up with is called `ec2selector`. Rather than forcing us to specify an AMI identifier, `ec2selector` allows us to declare a set of characteristics the AMI should comply with:

{% highlight python %}
from ec2selector import EC2Selector
EC2Selector('<aws access key>', '<aws secret key>').select(
    region_id=             'us-east-1', filters={
    'name_contains':       'ubuntu-10', 
    'owner_id':            '063491364108', # corresponds to 'alestic'
    'image_type':          'machine', 
    'virtualization_type': 'paravirtual',
    'architecture':        'x86_64', 
})
{% endhighlight %}
        
The previous code returns an image owned by [alestic](http://alestic.com) for an Ubuntu 10 machine with a 64-bit architecture hosted in the US East region. Today (April 2011), this corresponds to "ami-e450a28d", an AMD64 Ubuntu 10.10 Maverick image. In the future, a new version of Ubuntu 10 might be released, and this command will return the most updated image, regardless of its AMI identifier.

## Browse the repository

`ec2selector` has many other options [documented in its Github page](https://github.com/ff0000/ec2selector), including an interactive prompt to filter the list of available images by specifying one desired property after the other.
