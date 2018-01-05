---
title:  "My first public Ansible module rewrite"
date:   2018-01-05 22:45:00
categories: [ansible]
tags: [ansible, aws, sqs]
---

I've just completed my first attempt at re-writing and extending one of the core Ansible AWS modules and have submitted it back in to the upstream project for review and hopefully a merge.  Although I've written my own custom modules before, this was a lot tougher and much more fun!  I say tougher because I'm not only have to worry about my requirements but also everyone elses too.

The module I've re-written was the 'sqs_queue' module which is part of the core Ansible AWS modules.  It's been around a while (since v2.0) and it was developed against BOTO2.  This caused a problem as the behaviour I wanted was for the queues I built to be automatically tagged when built.  Unfortunately, this wasn't supported in BOTO2 and meant the module had to be re-written using BOTO3.

The process I followed was pretty much to take the existing module, upgrade all the commands and ensure I got a like for like copy working.  Then I added in some new functionality that I needed for tagging.

Whilst I was working on this module, I also added in support for FIFO SQS queues too, which was a feature request on the upstream Ansible project.  This turned out to be fairly easy; only requiring a couple of new parameters were passed at queue creation time.

Ansible are really pushing better quality of modules lately from v2.4 and following some of their guideline documents really helped.  These are some useful links that I used:

- [Guidelines for AWS modules](https://github.com/ansible/ansible/blob/devel/lib/ansible/modules/cloud/amazon/GUIDELINES.md)
- [AWS Working Group](https://github.com/ansible/community/tree/master/group-aws)
- [Ansible Dev guide](https://docs.ansible.com/ansible/latest/dev_guide/index.html)

Now, all I need to do is get my PR merged in and released with a future version.  In the mean time, I'll be pulling this in to the fork that Metapack deploy from to ensure we can take advantage of it right away!

Ansible PR can be found [here](https://github.com/ansible/ansible/pull/34526)
