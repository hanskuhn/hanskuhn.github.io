---
layout: post
title: BIND caching resolver with internal RRs
categories: [systems, networking]
tags: [DNS]
published: false
---

This needs to be written up.

http://guide.munin-monitoring.org/en/latest/tutorial/snmp.html#tutorial-snmp

https://support.pexip.com/hc/en-us/articles/204287889-Monitoring-Pexip-nodes-with-munin

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# cat /etc/munin/plugin-conf.d/snmp_communities
[snmp_*]
env.version 2
env.community s@ut-SNMP
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# munin-node-configure --snmpversion 2c --snmpcommunity STRINGHERE --snmp host.foo.com

# munin-node-configure --snmpversion 2c --snmpcommunity STRINGHERE --snmp host.foo.com --shell |sh -x

# systemctl restart munin-node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add host.foo.com to /etc/munin/munin.conf

NB that any SNMP host uses 'use_node_name **no**''.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# grep SAUT /etc/munin/munin.conf

[SAUT-server;server.foo.com]
    address 127.0.0.1
    use_node_name yes

[SAUT-network;host.foo.com]
    address 127.0.0.1
    use_node_name no
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


