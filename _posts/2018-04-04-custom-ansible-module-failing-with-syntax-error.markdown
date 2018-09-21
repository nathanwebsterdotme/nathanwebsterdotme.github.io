---
title:  "Custom Ansible Module Failing With Syntax Error"
date:   2018-04-04 19:30:00
categories: [devops]
tags: [ansible, modules]
---

Today has been spent debugging an issue that is quite hard to diagnose and easy to overlook.  One of those that makes no sense whatsoever, with awful error messages that don't help at all.

I have been testing some work which included a change to a custom Ansible module.  I knew that the custom module worked prior to this change so was able to rule out functionality problems pretty quickly.  The change to the custom module had changed it's name (for good reasons).  In this example, the change had renamed the Ansible module to **"custom-ansible-module"**

When testing the work (along with a bunch of other changes), I started to see an error similar to the following when we ran the playbook task that called the custom module:
```bash
fatal: [1.2.3.4 -> localhost]: FAILED! => {"changed": false, "module_stderr": "  File \"/home/local/nathanwebster/.ansible/tmp/ansible-tmp-1522766648.4-230460146388392/custom-ansible-module\", line 99\n    from ansible_module_custom-ansible-module import main\n                             ^\nSyntaxError: invalid syntax\n", "module_stdout": "", "msg": "MODULE FAILURE", "rc": 0}
```

This is all of the information Ansible gave me!  Helpful!

I tried a number of things to debug the problem but never got any more information than the above error message.  Running the module directly in Python, using the [steps detailed here](http://docs.ansible.com/ansible/latest/dev_guide/developing_modules_general.html#local-direct-module-testing) actually resulted in the module running successfully.  This added more confusion in to the mix as I knew the module worked OK and had the intended result, but invoking it from Ansible was not functioning.

I reverted the name change, and the module worked OK.  This led me to realise that the Ansible Module cannot have hyphens in the name, which was fairly obvious once it hit me!  I then changed the module name to "custom_ansible_module" and changed the corresponding task invocation and BOOM, it worked!

Lesson learned: no hyphens in custom Ansible modules!
