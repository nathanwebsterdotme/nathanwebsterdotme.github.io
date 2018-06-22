---
title:  "Testing WinRM Connections"
date:   2017-02-02 11:30:00
categories: [devops]
tags: [ansible, molecule, windows, winrm]
---

I've been playing around with [Molecule](https://github.com/metacloud/molecule) which is a great system for unit testing Ansible Roles.

Today, I had an issue when running `molecule test` for a Windows role I was working on.  Despite the problem I was experiencing being my own fault (I was running two instances of the same Windows Vagrant box, so the winrm ports were being automatically re-numbered), I found [this blog post](http://stevenmurawski.com/powershell/2015/06/need-to-test-if-winrm-is-listening/) from Steven Murawski detailing how to test if winrm is listening on the target machine directly from my Mac terminal.  This was a really useful post for helping me diagnose the issue.


#### Checks that the WinRM service is listening

`curl --header "Content-Type: application/soap+xml;charset=UTF-8" --header "WSMANIDENTIFY: unauthenticated" http://192.168.1.82:5985/wsman --data '&lt;s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope" xmlns:wsmid="http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd"&gt;&lt;s:Header/&gt;&lt;s:Body&gt;&lt;wsmid:Identify/&gt;&lt;/s:Body&gt;&lt;/s:Envelope&gt;'`

#### Checks that the local account can log in via WinRM using Basic Auth

`curl --header "Content-Type: application/soap+xml;charset=UTF-8" http://192.168.1.82:5985/wsman --basic  -u user:password --data '&lt;s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope" xmlns:wsmid="http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd"&gt;&lt;s:Header/&gt;&lt;s:Body&gt;&lt;wsmid:Identify/&gt;&lt;/s:Body&gt;&lt;/s:Envelope&gt;'`

Thanks Steven!

Courtesy of [Steven Murawski](http://stevenmurawski.com/powershell/2015/06/need-to-test-if-winrm-is-listening/)
