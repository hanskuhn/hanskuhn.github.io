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

role=target
initiatorname=iqn.2001-01.example.org:server
IP=10.10.0.25

The iSCSI initiator (client) lives on Ubuntu 18.04 on the EL-20.

role=initiator
initiatorname=iqn.1993-08.example.org:client
IP=10.10.0.30

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
devices can do serious damage. It goes without saying that you should use 
a dedicated network for any SAN.

On the initiator, there are two programs that are needed to mount the block
device; iscsid and open-iscsi. 

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bionic# systemctl enable --now iscsid 
bionic# systemctl enable --now  open-iscsi

bionic# iscsiadm -m discovery -t sendtargets -p freenas.example.org
freenas.example.org:3260,-1 iqn.2001-01.example.org:server:default-target

bionic# iscsiadm -m node # -o show is implicit, -T $node.name for specific target
# BEGIN RECORD 2.0-874
node.name = iqn.2001-01.example.org:server:default-target
node.tpgt = -1
node.startup = automatic
node.leading_login = No
...

bionic# iscsiadm -m node --login

bionic# iscsiadm -m session -P3 # -o show is implicit, -P3 for more detail
Target: iqn.2001-01.example.org:server:default-target (non-flash)
	Current Portal: 10.10.0.25:3260,1
	Persistent Portal: freenas.example.org:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.example.org:client
		Iface IPaddress: 10.10.0.30
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 1
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
...

bionic# lsblk --scsi
NAME HCTL       TYPE VENDOR   MODEL             REV TRAN
sda  0:0:0:0    disk FreeNAS  iSCSI Disk       0123 iscsi

bionic# disklabel GPT /dev/sda
bionic# parted 100% /dev/sda
bionic# mkfs.btrfs -f -L mybtrfspool  /dev/sda1
bionic# blkid /dev/sda1
/dev/sda1: UUID="b8087fc7-88c1-436f-a203-c05308af5db7" UUID_SUB="485f76cc-ef59-4d84-a6fa-7cc425dd9035" TYPE="btrfs" PARTLABEL="primary" PARTUUID="92e2e57a-5e88-4d7a-b2dd-c76a5116b7ac"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

Create an fstab entry on the intiator that mounts the block device at boot. 
Be sure that you use the _netdev flag so that the disk is mounted after the 
network is initialized.

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
UUID=b8087fc7-88c1-436f-a203-c05308af5db7 /media/freenas btrfs defaults,auto,_netdev 0 0
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

Configure LXD to use the BTRFS volume. Once you've spun up a few containers,
you can see the sub-volumes that BTRFS creates as it clones containers:

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bionic# btrfs subvol list /media/freenas
ID 257 gen 20586 top level 5 path containers
ID 258 gen 20586 top level 5 path containers-snapshots
ID 259 gen 56406 top level 5 path images
ID 260 gen 20586 top level 5 path custom
ID 261 gen 20586 top level 5 path custom-snapshots
ID 265 gen 56407 top level 257 path containers/c1-example-org
ID 268 gen 56400 top level 257 path containers/c2-example-org
ID 279 gen 56404 top level 259 path images/0b969c724698d2337eba00540128e26b17ff608a761c28fe7f91b9aef14c3a6e
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}
