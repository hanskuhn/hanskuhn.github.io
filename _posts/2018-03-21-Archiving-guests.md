---
layout: post
title: Archiving guests
---

As part of decommisioning a VM, I needed to archive the disk image which was qcow2 format. The VM 
wasn't sparse and had a lot of empty space. The solution was to fill the empty space with zeros 
after I deleted the usual cruft littering the filesystem.

{%highlight bash %}
guest# dd if=/dev/zero of=/zeros.dd
{%endhighlight %}

Wait for the disk to fill up and error out. Then I deleted the file before 
halting the guest.

{%highlight bash %}
guest# rm /zeros.dd
guest# halt -p
{%endhighlight %}

Next, I backed up the guest parameters and compressed the disk image.

{%highlight bash %}
host# virsh dumpxml > /root/guest.bogus.com_libvirt.xml
host# cd /var/lib/libvirt/images/guest.public.com/
host# mv disk_one.qcow2 disk_one_backup.qcow2
host# qemu-img convert -c -O qcow2 disk_one_backup.qcow disk_one.qcow2
{%endhighlight %}

I saw a 88% decrease in size! The original file was 57GB, and the final file was 6.5GB.

Finally, I offered up the guest directly from the host with python's SimpleHTTP server.

{%highlight bash %}
host# cd /path/to/disk_image/
host# sudo -Hu unprivuser nohup python -m SimpleHTTPServer 8822
{%endhighlight %}

The web server offers up a simple directory listing which might be a good idea to disable. 
I sent the URL, http://host.public.com:8822/, to the client and called it good.
