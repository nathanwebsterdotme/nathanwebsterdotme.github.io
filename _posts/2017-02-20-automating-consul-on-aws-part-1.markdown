---
title:  "Automating Consul in AWS | Part 1 | Automating Consul Server Deployments"
date:   2017-02-20 15:00:00
categories: [cloud, devops]
tags: [consul, ansible, aws, asg]
---

In this blog post, we'll cover some of the key parts of launching scalable Consul Clusters using AWS Auto Scaling Groups.  In part 2 of this series, we'll look at securing the cluster, backups and using Consul in Production.

I'll assume that you know enough about AWS infrastructure and how to launch EC2 instances using Launch Configurations and Auto Scaling Groups.  I'll also assume that you are installing Ansible 0.7.5, the latest version at the time of writing.  It's possible to completely automate all of the steps below using tools such as Ansible, which is what I've done in real life, but we'll keep this as simple as possible.  Get in touch if you are interested in how I did this!


#### Overview

In the example, we'll create a Consul Server cluster by following these four steps:

1. Deploy supporting AWS resources (IAM Role, Security Group).
2. Build an AMI.
3. Create a Launch Config
4. Deploy an ASG

First, we need to create our supporting resources like an IAM Role and Security Group in AWS that we can launch our Consul cluster with.  It's good practice to design and implement this up front, ensuring that your permissions only allow the minimum permissions needed to build the cluster (no allow-all rules!)
Secondly, we'll launch and configure an EC2 instance (installing Consul) and then create an AMI from it.  We'll use the AMI to create a launch config, attaching User Data that will customise the instances when they launch.
Finally, we'll create an Auto Scaling Group which will handle launching our EC2 instances for us.

So, let's get to it.


### 1. Deploy Supporting AWS Resources

Let's first look at the AWS resources we'll need to launch our cluster.

#### IAM Role
Our Consul Server nodes will need to query the AWS EC2 API to discover other Consul server nodes that it can join to create a cluster.  AWS recommends using IAM Roles to give permissions to EC2 instances.
To allow Consul to discover other Consul nodes, we need to allow the ec2:DescribeInstances permissions.  An example IAM Policy that would allow this is:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

#### Security Group
A Security Group is required to allow servers to communicate with each other.  The list of required ports required by Consul can be [found here](https://www.consul.io/docs/agent/options.html#ports).  Only one Security Group will be required as we'll launch all of the server instances attached to the same group.


#### Tagging
Tagging AWS resources is generally good practice to enable easy tracking of what is running in your AWS account and for reporting on costs.  We're also going to use EC2 Tags for discovering other Consul server nodes to create a cluster with so it's important to get our tags designed and implemented early.
In the example below, I use an EC2 tag pair of `"service": "consul-server"`.  This tag is applied to the Auto Scaling Group and propogated to all EC2 instances launched from it.


### 2. Build an AMI

We won't go in to the details of how to launch an EC2 instance here but you'll need an EC2 instance running Ubuntu and you will need to have installed Consul on to it.  Don't start the service just yet, we'll do this with our User Data later.
Once we have our server with Consun installed, there are a few configuration changes that we need to make to Consul.

### Consul Configuration File

Consul configuration files are processed in alphabetical order.  Let's create a consul config file here (assuming our install path is /opt/consul) `/opt/consul/consul.d/0-consul.json`
A basic configuration file with the options we need to launch a cluster might look like this:

```json
{
    "bootstrap-expect": 3,
    "server": true,
    "datacenter": "dc1",
    "data_dir": "/opt/consul/data",
    "log_level": "INFO",
    "enable_syslog": true,
    "leave_on_terminate": true,
    "bind_addr": "EC2_USER_DATA_BIND_ADDR",
 	  "retry_join_ec2": {
      "region": "eu-west-1",
  	  "tag_key": "service",
      "tag_value": "consul-server"
    }
}
```

There are a number of options available to us, [documented here](https://www.consul.io/docs/agent/options.html).  You'll need to use the ones to suit your needs but I'd like to discuss a few I've made use of in my example.


<br>
#### leave_on_terminate

[leave_on_terminate](https://www.consul.io/docs/agent/options.html#leave_on_terminate) sends a 'leave' message to the Consul cluster when it recieves a TERM signal, ensuring nodes leave the cluster gracefully.  This is useful for us when scaling down the number of nodes we have via AWS Auto Scaling Groups.

```json
{
  "leave_on_terminate": true
}
```

<br>
#### bootstrap_expect

The [bootstrap_expect](https://www.consul.io/docs/agent/options.html#bootstrap_expect) option allows us to specify the number of server nodes to wait for on launch before trying to create a cluster.  This helps us to prevent split-brain scenarios as we will avoid having a number of nodes launch and create clusters of 1 node.  In this example, we'll set this to 3, which is the number of nodes we'll launch in our Auto Scaling Group.
As each node launches, it will wait for 2 others to be joined before trying to create a cluster and elect a leader.  This is ideal when our three new nodes might take slightly different amounts of time to launch.

```json
{
  "bootstrap_expect": 3
}
```

<br>
#### bind_addr
The [bind_addr](https://www.consul.io/docs/agent/options.html#bind_addr) option allows us to explicitally specify which IP address of the server that Consul should use.  I like to do this to prevent any problems with Consul binding to other IP addresses or failing to launch if it can't decide which one to use.
As we are going to launch a number of instances from the same AWS AMI, we can't set this value before we launch the instance, so we have to template this variable and change this after we launch our live AWS EC2 instance.  We can do this easily with EC2 User Data (more on this later).  For now, let's set this variable to something we can target easily later, such as "EC2_USER_DATA_BIND_ADDR".

```json
{
  "bind_addr": "EC2_USER_DATA_BIND_ADDR"
}
```

<br>
#### retry_join_ec2

In version 0.7.1 of Consul, Hashicorp introduced the `retry_join_ec2` configuration option.  This allows us to configure Consul to automatically discover other EC2 nodes via Tags, rather than having to provide a list of IP addresses.  Again, this is ideal when launching a set of nodes in an Auto Scaling Group, as they will all discover each other as long as our tags are consistent.
Here, we tell the Consul server to look for other instances with the tag key:value pair of `"service": "consul-server"` in the eu-west-1 region.  The Consul instance will query the AWS API every 30 seconds until it successfully discovers and joins another instance running Consul Server. s

```json
{
  "retry_join_ec2": {
    "region": "eu-west-1",
    "tag_key": "service",
    "tag_value": "consul-server"
  }
}
```

As we created an IAM Role earlier to enable the correct permissions for EC2 discovery, we don't need to specify our AWS keys here as our EC2 instance will have intrinsic permissions to do so.  If you did want to hard-code your AWS api keys instead of using IAM roles (not recommended) then you can do so like this:

```json
{
  "retry_join_ec2": {
    "region": "eu-west-1",
    "tag_key": "service",
    "tag_value": "consul-server",
    "access_key_id": "12345",
    "secret_access_key": "56789"
  }
},
```


#### Bake the AMI
Once the configuration file is in place, we'll bake our AMI with the Consul service stopped and not enabled.  We do this so that we can change our hostname after launching our instances as well as replacing the `bind_addr` variable in our config file before starting Consul.  We will handle these changes in the EC2 User Data.


### 3. Create a Launch Config.

We won't cover how to create a Launch Config in AWS but we will look at the User Data script we need to run.

#### User Data

As we create a generic AMI of our consul server, we need to do some basic customisation when our instances launch.  This includes setting the hostname, replacing the consul bind IP address in the config file and starting the consul service.  We can do this easily with AWS User Data using the following script:

```bash
#!/bin/bash

# This script replaces the hostname of the server
# and then changes the bind address in consul config with the local IP address.
# Finally, it starts the consul service.

internalIP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
instanceID=$(curl http://169.254.169.254/latest/meta-data/instance-id)
hostname="consul-${instanceID#*-}"

hostnamectl set-hostname $hostname
cat << EOF >> /etc/hosts
127.0.0.1 $hostname
EOF

sed -i -e "s/EC2_USER_DATA_BIND_ADDR/$internalIP/g" /path/to/consul/consul.d/0-consul.json

systemctl start consul
```
This will ensure each instace is customised on boot up enough for Consul to work as we want.

### 4. Launch our ASG

Finally, we launch our Auto Scaling Group.  As we set our `bootstap_expect` variable to 3, we need to launch at least three instances to see a cluster be created.


<br>
<br>

#### Credits

Thanks to [this great blog post from Sky Bet](http://engineering.skybettingandgaming.com/2016/05/05/aws-and-consul/) which gave me initial guidance when starting out.
