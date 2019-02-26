---
layout: post
title: SSH restricted commands with rsync
categories: [systems, security]
tags: [SSH]
published: false
---

On one of the systems I admin, there's a synchronization script that copies an htdigest password file.
If users update their credential through a web portal, the update gets
propagated to different web services that use the authentication data.

I found that the script wasn't working while troubleshooting it, decided
to reduce the attack surface for a decidedly bad idea -- copying credentials
around rather than use a network service like LDAP for authentication.

Because there are two files (htdigest and htgroup), I had to use a
feature of SSH that was new to me. Normally you specify the 
command you want to allow in the remote user's authorized_keys
file. The 'command' declaration is prepended before the public key.

The first challenge I had was that the host I was copying the files from
was freebsd and rsync is not part of the base system. That means the binary
lives in /usr/local. I naturally used /usr/bin/rsync as the path which was
never going to work. 

After I solved that, I found that the command wasn't complete and I wasn't sure
what command was being executed on the remote side. Luckily rsync allows you to 
debug 

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/usr/bin/rsync -vlpPtgoHx remote.host.com:/usr/local/www/apache24/htgroup .
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

Adding another '-v' flag showed me the remote command being executed. Here's what it looked
like:

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ssh remote.host.com rsync --server --sender -vvlHogtpxe.Lsfx . /usr/local/www/apache24/htgroup
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

There are two paths that need to be copied with the password syncing
script, so I had to rely on ${SSH_ORIGINAL_COMMAND} to permit more than
one command to be executed on the remote side.

Here's what the final ssh authorized_keys file looks like on remote.home.com:

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
command="/usr/local/bin/rsync --server --sender -vvlHogtpxe.Lsfx . ${SSH_ORIGINAL_COMMAND//* \//\/}",no-pty,no-agent-forwarding,no-port-forwarding ssh-rsa AAAA...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{%endhighlight %}

