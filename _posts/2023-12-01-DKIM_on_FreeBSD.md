---
layout: post
title: DKIM on FreeBSD
categories: [systems]
tags: [sendmail, DKIM, freebsd]
published: true
---

Google announced that DKIM will be mandated in February 2024, so I started
investigating how to deploy this for the small number of domains that I host.

I'm using FreeBSD 13 with Sendmail for SMTP services.

OpenDKIM is the usual open source solution for validating incoming messages
and signing outgoing messages.

I use pkgng, so let's install OpenDKIM:

{%highlight bash %}
$ pkg install opendkim
{%endhighlight %}

The package maintainers have a few notes that are somewhat useful, if
not complete:

{%highlight bash %}
Message from opendkim-2.10.3_16:

--
In order to run this port, write your opendkim.conf and:

if you use sendmail, add the milter socket `socketspec' in
/etc/mail/<your_configuration>.mc:

INPUT_MAIL_FILTER(`dkim-filter', `S=_YOUR_SOCKET_SPEC_, F=, T=R:2m')

or if you use postfix write your milter socket `socketspec' in
/usr/local/etc/postfix/main.cf:

smtpd_milters = _YOUR_SOCKET_SPEC_


And to run the milter from startup, add milteropendkim_enable="YES" in
your /etc/rc.conf.
Extra options can be found in startup script.

Note: milter sockets must be accessible from postfix/smtpd;
  using inet sockets might be preferred.
{%endhighlight %}

Let's configure OpenDKIM to use some sane defaults:

{%highlight bash %}
# cd /usr/local/etc/mail
# vi opendkim.conf
{%endhighlight %}

Change these values in `opendkim.conf`:

{%highlight bash %}
Canonicalization relaxed/relaxed
KeyTable         refile:/usr/local/etc/mail/opendkim.keytable
LogWhy           yes # disable in prod
MilterDebug      1   # disable in prod
ReportAddress    "DKIM Error Postmaster" <postmaster@example.com>
#Selector        20231205
SigningTable     refile:/usr/local/etc/mail/opendkim.signingtable
Socket           local:/var/run/dkim/opendkim.sock
Syslog           Yes
SyslogSuccess    Yes
UserID           mailnull:mailnull
{%endhighlight %}

## Generate keypair

Let's generate a key to be stored in `/var/db/dkim/` using '/usr/local/sbin/opendkim-genkey'.

I'm going to use the following values for my keypair that will be used for signing outgoing
email. The public key is stuffed into the DNS using the format 

{%highlight bash %}
selector._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=VERYLONGSTRING"
{%endhighlight %}

{%highlight bash %}
selector:        20231205 # date is a common choice
domain:          example.com
keylength:       2048
privatekey_dir:  /var/db/dkim/example.com/
{%endhighlight %}

Let's make a keypair!
{%highlight bash %}
# opendkim-genkey -r -s 20231205 -d example.com -b 2048 --notestmode -D /var/db/dkim/example.com
{%endhighlight %}

This command created two files  in `/var/db/dkim/example.com`. The private key is **20231205.private**
and the public key is **20231205.txt**.

Let's tell OpenDKIM about our private key and what domain it is associated with
by creating entries in the signing table and keytable:

{%highlight bash %}
# cat /usr/local/etc/mail/opendkim.keytable

20231205._domainkey.example.com example.com:20231205:/var/db/dkim/example.com/20231205.private
{%endhighlight %}

{%highlight bash %}
# cat /usr/local/etc/mail/opendkim.signingtable

*@example.com 20231205._domainkey.example.com
{%endhighlight %}

WARNING: These commands work on my setup to fix the permissions. Please
don't blindly copy these commands.

{%highlight bash %}
# chgrp mailnull /usr/local/etc/mail/opendkim.*
# chmod o-rwx /usr/local/etc/mail/opendkim.*
# chown -R mailnull:mailnull /var/db/dkim
{%endhighlight %}

Now let's enable OpenDKIM, and ask it to run as a non-priv user that plays
nicely with Sendmail:

{%highlight bash %}
# sysrc milteropendkim_enable="YES"
# sysrc milteropendkim_uid="mailnull"
# sysrc milteropendkim_cfgfile="/usr/local/etc/mail/opendkim.conf"
{%endhighlight %}

Start milter-opendkim service 

{%highlight bash %}
# service milter-opendkim start
# service milter-opendkim status
{%endhighlight %}

### Publish the pubkey in DNS

The public key was written to /var/db/dkim/example.com when
we generated our keypair:

{%highlight bash %}
$ cat /var/db/dkim/example.com/20231205.txt
{%endhighlight %}

The pubkey is copy-paste-ready for BIND. Publish it in
your zonefile, increment the serial number, and reload
your zonefile.

Then check your key:

{%highlight bash %}
$ dig TXT 20231205._domainkey.example.com. +short
{%endhighlight %}

### Sendmail

I need to update my sendmail.cf, which I keep in RCS for version control
as this was setup back when dinosaurs roamed the Earth:

{%highlight bash %}
# cd /etc/mail
# co -l example.com.mc
# vi example.com.mc
{%endhighlight %}

Here's the two lines that need to be added to your .mc file:

{%highlight bash %}
INPUT_MAIL_FILTER(`dkim-filter', `S=local:/var/run/dkim/opendkim.sock, F=T, T=R:2m')dnl
define(`confINPUT_MAIL_FILTERS', `dkim-filter')dnl
{%endhighlight %}

{%highlight bash %}
# make  # ask m4 to do some magic to generate the .cf file
# cp sendmail.cf sendmail.cf.bak # just in case
# cp example.com.cf sendmail.cf
# ci -u example.com.mc
# service sendmail restart
{%endhighlight %}

## Testing

  - Send email to `check-auth@verifier.port25.com` which will reply with a report validating DKIM, SPF, rDNS
  - If you prefer the web, use <https://dkimvalidator.com/> to see if the key in DNS can validate signatures from from the milter
  - Check that the milter is handling incoming email correctly by looking in `/var/log/maillog` or mail headers of received msgs. Look for Authentication-Results header!

## Operations

How often should I publish a new keypair? 

Should I expire and publish private keys periodically? (cf. <https://blog.cryptographyengineering.com/2020/11/16/ok-google-please-publish-your-dkim-secret-keys/>)

What monitoring is possible to ensure that deliverability is good? DMARC
reports are probably a start, but jeez.

### DMARC
 
Once DKIM is working, you'll probably want to publish a DMARC RR like this: 

**rua=** Set the email address where you want DMARC aggregate reports delivered

**ruf=** Set the email address where you want DMARC failure reports delivered

**sp=none** TODO

**aspf=r** SPF is relaxed

**adkim=r** DKIM is relaxed

**fo=1** Send DMARC failure reports to ruf=

{%highlight bash %}
_dmarc.example.com TXT v=DMARC1; p=none; rua=mailto:dmarc@example.com; ruf=mailto:dmarcfail@example.com; sp=none; aspf=r; adkim=r; fo=1;
{%endhighlight %}

References:

 - <https://www.dan.me.uk/blog?post=7524cdb7315070816d1a15286b63346cbeded4fa>
 - <http://www.elandsys.com/resources/sendmail/dkim.html>
 - <https://petermolnar.net/article/howto-spf-dkim-dmarc-postfix/>
 - <https://www.skelleton.net/2015/03/21/how-to-eliminate-spam-and-protect-your-name-with-dmarc/>
 - <https://www.web-workers.ch/index.php/2019/10/21/how-to-configure-dkim-spf-dmarc-on-sendmail-for-multiple-domains-on-centos-7/>
 - <https://knowledge.validity.com/hc/en-us/articles/222438487-DKIM-signature-header-detail>
 - <https://www.mail-tester.com/>
 - <https://dkimvalidator.com/>
 - <https://dmarcian.com/dkim-inspector/>
 
