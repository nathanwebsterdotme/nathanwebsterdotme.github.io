---
title:  "Jinja default (omit) filter for Ansible"
date:   2017-05-15 15:35:00
categories: [ansible]
tags: [jinja, ansible, devops]
---

I love the ```default (omit)``` jinja filter in Ansible.

I'm currently writing a lot of roles in the Ansible code I'm currently working on.  A number of these are for managing AWS resources which sometimes have optional parameters.  The "default (omit)"" filter allows me to create roles that can handle these option paramters without having to check if they exists using conditionals.

Let's take the 'ec2' ansible module for example.  I'm using a simplified set of arguments here, but let's assume that the following is true:

```
# Mandatory Arguments:
ec2_vars:
  name:  test123
  ami:  ami-12345
  security_group:  sg-12345

# Optional Arguments:
ec2_vars:
  iam_role_name:  testIAM123

```

If I didn't use the filter, I'd have to introduce a boolean variable called "iam_role" and write a role similar to:

```
### Sample Vars ###
# vars:
#   ec2_vars:
#   name:  test123
#   ami:  ami-12345
#   security_group:  sg-12345
#   iam_role: false
#   iam_role_name: testIAM123   # Used if iam_role set to true

tasks:
- name: Launch an EC2 instance without IAM role
  ec2:
    name: "\{\{ ec2_vars.name \}\}"
    image: "\{\{ ec2_vars.ami \}\}"
    group: "\{\{ ec2_vars.security_group \}\}"
  when: ec2_vars.iam_role == false

- name: Launch an EC2 instance with an IAM role
  ec2:
    name: "\{\{ ec2_vars.name \}\}"
    image: "\{\{ ec2_vars.ami \}\}"
    group: "\{\{ ec2_vars.security_group \}\}"
    instance_profile_name: "\{\{ ec2_vars.iam_role.name \}\}"
  when: ec2_vars.iam_role == true

```

I've seen roles written like this.  Obviously, this isn't good as we're duplicating code unecessarily.

However, with our ```default(omit)``` filter, our role would look like this:

```
### Sample Vars ###
# vars:
#   ec2_vars:
#   name:  test123
#   ami:  ami-12345
#   security_group:  sg-12345
#   iam_role_name: testIAM123   # Used if iam_role set to true

tasks:
- name: Launch an EC2 instance with an IAM role
  ec2:
    name: "\{\{ ec2_vars.name \}\}"
    image: "\{\{ ec2_vars.ami \}\}"
    group: "\{\{ ec2_vars.security_group \}\}"
    instance_profile_name: "\{\{ ec2_vars.iam_role.name | default (omit) \}\}"

```

This makes the ```instance_profile_name``` argument optional and if no value is provided, we'll omit sending any data for it.

I'm using this quite often now, especially for AWS Ansible modules.  I think it's a really useful tool for writing re-usable roles that can be made to handle many different use cases!
