---
layout: post
title: Archiving guests
---

As part of decommisioning a VM, I needed to archive the disk image. The VM wasn't sparse and had a lot of empty 
space. The solution was to fill the empty space with zeros after I deleted the usual cruft littering the 
filesystem.

{%highlight bash %}
guest# dd if=/dev/zero of=/zeros.dd
guest# rm /zeros.dd
guest# halt -p
host# virsh dumpxml > /root/guest.bogus.com_libvirt.xml
host# cd /var/lib/libvirt/images/guest.bogus.com/
host# mv disk_one.qcow2 disk_one_backup.qcow2
host# qemu-img convert -c -O qcow2 disk_one_backup.qcow disk_one.qcow2
{%endhighlight %}

I saw a 88% decrease in size. The original file was 57GB, and the final file was 6.5GB!
