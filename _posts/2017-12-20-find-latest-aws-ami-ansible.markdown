---
title:  "Find the latest AWS AMI with Ansible"
date:   2017-12-20 18:30:00
categories: [ansible]
tags: [ansible, windows, ec2]
---

I am working on a project that builds on top of Windows Server 2016.  One of the requirements I have is to ensure all Windows updates are installed on Windows Server before I install the application.

The first method I used for this was to have an Ansible role that handled installing Windows Updates before doing anything else.  This used the [win_updates](https://docs.ansible.com/ansible/latest/win_updates_module.html) module and worked fine.  However, I noticed this took a long amount of time to complete, and it kept taking longer and longer with every release.

It turns out this was due to us using a set AWS EC2 AMI ID in the code, and always launching from that.  As this AMI got 'older' in terms of Windows Updates, we had more to install each time we deployed.

To fix this, I wrote a quick role to go to find the latest version of the AMI and use this to build from.  This would ensure I was always building from the latest Windows Server 2016 AMI provided by AWS, which had all the latest Windows updates installed and tested by AWS.

The role looks like this:

```
---
- name: Find latest AWS EC2 AMI by name
  ec2_ami_find:
    name: "{{ ec2_latest_ami.name_pattern }}"
    region: "{{ aws_region }}"
    sort: "{{ sort_type | default ('creationDate') }}"
    owner: "{{ ec2_latest_ami.owner | default (omit) }}"  # Optional owner filter
    sort_order: "{{ sort_order | default ('descending') }}"
    sort_end: 1
  register: ec2_ami

- name: set ec2 ami ID as fact
  set_fact:
    ec2_instance_ami: "{{ ec2_ami.results.0.ami_id }}"

```

This change cut down the infrastructure build time from around 1 hour to around 11 minutes - a big improvement for the project!
