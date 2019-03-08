---
layout: post
title: Remote Block Device with iSCSI
categories: [systems, storage]
tags: [FreeBSD]
published: false
---

I have a small server (Gigabyte EL-20) intended for IoT so it lacks a reasonable 
amount of storage.  Time to investigate network storage so I can deploy some 
LXD containers on it!

The iSCSI target (server) lives on FreeNAS which is just FreeBSD crippled up
with a web gui.

The iSCSI initiator (client) lives on Ubuntu 18.04 on the EL-20.

The iSCSI service uses port 3260 by default. Keep this in mind if you have 
host or network firewalls and can't connect your initiator to your target.

iSCSI expects the initiator to have a specially formatted name. The name is
stored in /etc/iscsi/initiatorname.iscsi. You can generate a unique name
with the iscsi-iname utility.

Something like this will get you running, but remember that you'll need
to give this name to the portal so it is authorized to access the target.

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bionic# mv /etc/iscsi/initiatorname.iscsi /etc/iscsi/initiatorname.iscsi.dist
bionic# echo "InitiatorName=`/sbin/iscsi-iname`" > /etc/iscsi/initiatorname.iscsi
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

Configuration for the target for FreeBSD is stored in /etc/ctl.conf.
I configured the portal using the FreeNAS gui.

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
freenas# ctladm lunlist
freenas# ctladm devlist -v
freenas# ctladm portlist -v
freenas# ctladm islist -v

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

If there are credentials to access the target, you'll need to add them to 
/etc/iscsi/iscsid.conf. It's also possible to limit connections to an IP 
block. It's best to do both given that anyone with access to your block 
devices can do serious damage.

On the initiator, there are two programs that are needed to mount the block
device; iscsid and open-iscsi. 

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bionic# systemctl enable --now iscsid 
bionic# systemctl enable --now  open-iscsi

bionic# iscsiadm -m discovery -t sendtargets -p nas.home.lan
nas.home.lan:3260,-1 iqn.2001-01.lan.home:freenas:default-target

bionic# iscsiadm -m node # -o show is implicit, -T $node.name for specific target
# BEGIN RECORD 2.0-874
node.name = iqn.2001-01.lan.home:freenas:default-target
node.tpgt = -1
node.startup = automatic
node.leading_login = No
...

bionic# iscsiadm -m node --login

bionic# iscsiadm -m session -P1 # -o show is implicit, -P1 for more detail
Target: iqn.2001-01.lan.home:freenas:default-target (non-flash)
	Current Portal: 192.168.0.25:3260,1
	Persistent Portal: nas.home.lan:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.lan.home:wren
		Iface IPaddress: 192.168.0.30
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 1
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

Create an fstab entry on the intiator that mounts the block device at boot. 
Be sure that you use the _netdev flag so that the disk is mounted after the 
network is initialized.

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/dev/sda1 /media/freenas btrfs defaults,auto,_netdev 0 0
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

