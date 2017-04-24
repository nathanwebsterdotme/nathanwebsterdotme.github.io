---
title:  "Set Fact as list in Ansible"
date:   2017-04-24 12:50:00
categories: [ansible]
tags: [ansible, devops]
---

I've just been battling with this one in Ansible so thought I'd post up the solution.

Problem: I want to use the ```set_fact``` module to create a list in Ansible.  I am currently trying to use the ```ec2_asg_facts``` to get a list of instance_ids.  The instance_ids are returned as a larger set of results, so I need to loop through the results of the ```ec2_asg_facts``` module and grab the instance_ids.

Solution:
The following code did the job!

```
- name: Look up AWS ASG by name
  ec2_asg_facts:
    region: "{{ region }}"
    name: "{{ ec2_asg_name }}"
  register: ec2_asg_facts_results

- name: Create list of instance_ids
  set_fact:
    ec2_asg_instance_ids: "{{ ec2_asg_facts_results.results[0].instances | map(attribute='instance_id') | list }}"

```