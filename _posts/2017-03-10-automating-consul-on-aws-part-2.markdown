---
title:  "Automating Consul in AWS | Part 2 | Securing Consul"
date:   2017-03-15 12:00:00
categories: [consul, aws]
tags: [consul, security, aws, devops]
---

In Part 2 of this Automating Consul in AWS series, we're going to focus on securing our Cluster.  We'll look at some of the Consul features that can help us secure our cluster like gossip encryption, TLS authentication and disabling remote exec.  I planned to incorporate ACL rules here too, but as Consul 0.8 has just been released, with improved ACL coverage, we'll cover that in a seperate post.


### Consul Encryption
We want to ensure that gossip communication between our servers and agents is secure.  This is especially true in scenarios where you are running Consul across different datacenters in distinct locations.
> TLS is used to secure the RPC calls between agents, but gossip between nodes is done over UDP and is secured using a symmetric key. (https://www.consul.io/docs/agent/encryption.html)

#### [Gossip Encryption](https://www.consul.io/docs/agent/encryption.html)
Gossip encryption is easily enabled.  First, generate a gossip encryption key using ```consul keygen``` then add the following to your server and agent config file:
```
{
  "encrypt": "encryptionkey"
}
```

#### [TLS](https://www.consul.io/docs/agent/encryption.html)
As we said before, TLS is used to secure the RPC calls between agents.  We can enable checks on both outbound and inbound connections.  All of our Consul server and agent nodes must use certificates signed by the same Certificate Authority.
As per the Consul website, there is a tutorial on how to create certificates [here](http://russellsimpkins.blogspot.com/2015/10/consul-adding-tls-using-self-signed.html)

Once we have the certificates on our server, we can enable the use of them in our config file as follows:
```
{
  "ca_file": "path/to/ca.cert",
  "cert_file": "path/to/consul.cert",
  "key_file": "path/to/consul.key"
}
```
By providing the certificates, TLS will be automatically enabled but we need to specify when we want to use it.

```
{
  "verify_incoming": true,
  "verify_outgoing": true
}
```

'verify_incoming' needs to be enabled on the server nodes.  This will ensure that servers verify the authenticity of all incoming connections and will prevent any non-TLS connections from being established.
'verify_outgoing' should be enabled for both the client and server nodes.  This will ensure that the authenticity of the outbound connections are verified.
More information on Encryption can be found on the Consul website [here](https://www.consul.io/docs/agent/encryption.html)


### [Consul Exec](https://www.consul.io/docs/commands/exec.html)
Consul provides an built-in mechanism for remote execution called "exec".  With the right ACL permissions, it's possible to run remote commands on any number of nodes joined to the cluster.  For me, that's a massive attack vector and needs to be disabled.  Fortunately, this is easy to do via the server config.  Just add the following line to the ```0-consul.json``` file from Part 1.  (note - this is disabled by default in v0.8 and above)

```
{
    "disable_remote_exec": true
}
```

#### Credit
This post builds on Oliver Mauras' [excellent Securing Consul blog post](https://www.mauras.ch/securing-consul.html#prevent-rogue-nodes-joining-the-cluster)