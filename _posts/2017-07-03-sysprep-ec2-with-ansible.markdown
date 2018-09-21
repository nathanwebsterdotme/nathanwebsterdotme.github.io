---
title:  "Sysprep Windows 2016 EC2 with Ansible"
date:   2017-07-03 18:30:00
categories: [devops]
tags: [ansible, windows, ec2]
---

Windows instances can be a funny beast and don't always play ball with your automation intentions.  I'm currently working on automating the build of a cluster of Microsoft AD servers and came across a problem when trying to join the second DC to the first.
The problem? I had built these instances from the same custom AMI and hadn't sysprepped them!

From previous playing around, I knew this could be an issue as the Sysprep action shuts the EC2 instance down and therefore the Ansible connection errors.  We need a way around this!

### Asynchronous Ansible
Step in Async tasks in Ansible!

By using Asynchronous task execution in Ansible, I can basically 'fire and forget' the sysprep action which will keep my playbook execution alive whilst the Windows instance is sysprepped and shut down, ready for an AMI to be created from it.  As the instance has been sysprepped, a new SID will be generated on launch meaning my nodes can join the Active Directory domain with no problems.

I decided to do a 'fire and forget' async task by setting the 'poll' value to 0 as follows.  Note, this is the correct task for Windows Server 2016 on AWS EC2.

```yaml
---
- name: Schedule EC2Launch to run on next boot
  win_shell: 'C:\ProgramData\Amazon\EC2-Windows\Launch\Scripts\SysprepInstance.ps1'
  async: 120   # I'm not too sure what this does when 'poll' is 0.
  poll: 0
```

So now I have my Sysprep action running, this will take a few minutes and will shut the instance down gracefully.  Before I move on to the AMI creation task though, I want to ensure this has completed succesfully so I added the following task on the Ansible localhost:

```yaml
- name: Run the sysprep command on the target
  hosts: ec2_instance

 tasks:
  - name: Run sysprep on EC2 instance asynchronously
    win_shell: 'C:\ProgramData\Amazon\EC2-Windows\Launch\Scripts\SysprepInstance.ps1'
    async: 120   # I'm not too sure what this does when 'poll' is 0!
    poll: 0

- name: Build AMI of instance
  hosts: localhost

  tasks:
  - name: Query AWS API and wait until instance is in 'stopped' state
    ec2_remote_facts:
      filters:
        instance-id: 'i-1234567890'
    register: reg_ec2_remote_facts
    until: reg_ec2_remote_facts.instances[0].state == "stopped"
    retries: 20
    delay: 15

  # Build AMI tasks...

```

More information on Asynchronous tasks can be found here: https://docs.ansible.com/ansible/playbooks_async.html
