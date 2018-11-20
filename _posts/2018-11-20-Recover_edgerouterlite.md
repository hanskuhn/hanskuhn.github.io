---
layout: post
title: Recover Ubiquiti Edgerouter Lite 3
---

I planned to spend my evening after work configuring an IoT VLAN on my
home network so I can play around with Mozilla's IoT Gateway project.
Things didn't go as planned. After my first couple of changes to add
a VLAN to a subinterface, I tried to commit and got a read-only error.
Rather than debug the problem, I figured a reboot might be needed because
perhaps I upgraded the OS and didn't have time to reboot and, and, I don't 
really know what I was thinking.

It turns out it was a dying USB thumb drive in my ERL-3. I grabbed gddrescue
and copied the filesystem for /dev/sdX2, the 1.6G partition where the volatile
system state is stored. I was most interested in /config/config.boot which
because I didn't have a current version of that file saved anywhere.

Here's the filesystems on my ERL-3

{%highlight bash %}
admin@gw:~$ lsblk
NAME      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda         8:0    1   7.5G  0 disk
|-sda1      8:1    1   142M  0 part
`-sda2      8:2    1   1.6G  0 part
loop8       7:8    0  77.3M  1 loop
mtdblock0  31:0    0   512K  1 disk
mtdblock1  31:1    0   512K  1 disk
mtdblock2  31:2    0    64K  1 disk
{%endhighlight %}

Note sda2 is 1.6G. That's the one I want.

On my Ubuntu laptop I installed gddrescue and plugged the failing thumb drive
into the laptop. The thumbdrive came up as /dev/sdb.

{%highlight bash %}
laptop# mkdir /tmp/usb_rescue
laptop# ddrescue /dev/sdb /tmp/usb_rescue/disk_image /tmp/usb_rescue/logfile
laptop# e2fsck -y /tmp/usb_rescue/disk_image
laptop# mount -o loop /tmp/usb_rescue/disk_image /mnt
{%endhighlight %}

I copied /mnt/w/config/config.boot to a webserver so I could later load
the config.boot file over HTTP.

Now I needed to bootstrap the ERL-3 with a new system image. Ubiquiti has a tool
called emrk. It lives at http://packages.vyos.net/tools/emrk/

Format an 8G+ USB thumb drive with FAT32 filesystem and MBR disklabel. I used my 
Macbook Air and DiskUtility.app for this step. Copy emrk-0.9c.bin to the 
newly-formatted thumbdrive and insert it into your ERL-3. Plug the power into 
your router and watch it boot on your console cable. This process will use 
your LAN's DHCP server and Internet connection if available, so make sure you 
plug the router's eth0 interface into the LAN. It will use this to download 
the latest OS from Ubiquiti's website.

Now we want to run the bootloader:

{%highlight bash %}
router# fatload usb 0 $loadaddr emrk-0.9c.bin
{%endhighlight %}

After it boots you should see a message like 15665511 bytes read.

Now you can load the rescue image:

{%highlight bash %}
router# bootoctlinux $loadaddr
{%endhighlight %}

The final step is to reinstall the Edgemax OS:

{%highlight bash %}
router# emrk-reinstall
{%endhighlight %}

This process will prompt you several times and you will need to provide 
the URL to the latest version the Edgemax OS.

http://dl.ubnt.com/firmwares/edgemax/v1.7.0/ER-e100.v1.7.0.4783374.tar

Unplug the router from any LAN until you have an initial configuration.

{%highlight bash %}
host# cd /path/to/disk_image/
host# sudo -Hu unprivuser nohup python -m SimpleHTTPServer 8822
{%endhighlight %}

