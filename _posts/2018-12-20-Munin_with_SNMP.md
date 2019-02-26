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

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# cat /etc/munin/plugin-conf.d/snmp_communities
[snmp_*]
env.version 2
env.community my$NMP-$3cr3t
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# munin-node-configure --snmpversion 2c --snmpcommunity my$NMP-S3cr3t --snmp host.foo.com

# munin-node-configure --snmpversion 2c --snmpcommunity my$SNMP-S3cr3t --snmp host.foo.com --shell |sh -x

# systemctl restart munin-node
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

Add host.foo.com to /etc/munin/munin.conf

NB that any SNMP host uses 'use_node_name **no**''.

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# grep SAUT /etc/munin/munin.conf

[SAUT-server;server.foo.com]
    address 127.0.0.1
    use_node_name yes

[SAUT-network;host.foo.com]
    address 127.0.0.1
    use_node_name no
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}


