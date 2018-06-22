---
title:  "Ansible fails with ansible.vars.unsafe_proxy.AnsibleUnsafeText object"
date:   2017-05-31 18:01:00
categories: [devops]
tags: [ansible, windows, iis]
---

Sometimes, we all have those days where you aare convinced your code is completely right and despite checking it over and over, you just can't decipher what's wrong with it.  Today was one of those days.

One of my ansible tasks was failing with this error message:
```
TASK [win_iis_webapppool : Create / Delete Win IIS App Pool] *******************
task path: /path/to/ansible/roles/win_iis_webapppool/tasks/main.yml:3
fatal: [win-iis-webapppool]: FAILED! => {
    "failed": true,
    "msg": "the field 'args' has an invalid value, which appears to include a variable that is undefined. The error was: 'ansible.vars.unsafe_proxy.AnsibleUnsafeText object' has no attribute 'name'\n\nThe error appears to have been in '/Users/nwebster/git/devops-infrastructure/ansible2/roles/win_iis_webapppool/tasks/main.yml': line 3, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: Create / Delete Win IIS App Pool\n  ^ here\n"
}
```

This is a role I'm working on to Create / Delete one or more IIS App Pools on a Windows machine.

This is the role:

```
---
- name: Create / Delete Win IIS App Pool
  win_iis_webapppool:
    name: "{{ item.name }}"
    state: "{{ item.state | default ('started') }}"
    attributes: "{{ item.attributes | default (omit) }}"
  with_items: win_iis_app_pools
```

I configured the role with a variable tree similar to:

```
win_iis_app_pools:
- name: 'win-iis-webapppool-test'
  state: 'started'
  attributes: 'managedRuntimeVersion:v4.0'
```

Can you see the problem and why it was failing?  If you can, well done you!

The issue is with the ```with_items: win_iis_app_pools``` line in the task.  As of Ansible 2.2, this will now throw an error, albeit a pretty useless one!

This was the fix:

```
---
- name: Create / Delete Win IIS App Pool
  win_iis_webapppool:
    name: "{{ item.name }}"
    state: "{{ item.state | default ('started') }}"
    attributes: "{{ item.attributes | default (omit) }}"
  with_items: "{{ win_iis_app_pools }}"
```

Hopefully this might help someone else save some time and their head from banging against the table!
