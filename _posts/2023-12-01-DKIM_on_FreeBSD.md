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

I need to update my sendmail.cf, which I keep in RCS for version control
as this was setup back when dinosaurs roamed the Earth:


{%highlight bash %}
$ cd /etc/mail
$ co -l hiroshima.bogus.com.mc
$ vi hiroshima.bogus.com.mc
{%endhighlight %}

Here's the two lines that need to be added to your .mc file:

{%highlight bash %}
INPUT_MAIL_FILTER(`dkim-filter', `S=local:/var/run/dkim/opendkim.sock, F=T, T=R:2m')dnl
define(`confINPUT_MAIL_FILTERS', `dkim-filter')dnl
{%endhighlight %}

{%highlight bash %}
$ make  # ask m4 to do some magic to generate the .cf file
$ cp sendmail.cf sendmail.cf.bak # just in case
$ cp hiroshima.bogus.cf sendmail.cf
{%endhighlight %}

Now let's enable OpenDKIM, and ask it to run as a non-priv user that plays
nicely with Sendmail:

{%highlight bash %}
$ cd /etc
$ vi rc.conf
{%endhighlight %}

Here's the lines to be added to rc.conf:

{%highlight bash %}
milteropendkim_enable="YES"
milteropendkim_uid="mailnull"
milteropendkim_cfgfile="/usr/local/etc/mail/opendkim.conf"
{%endhighlight %}

Let's configure OpenDKIM to use some sane defaults

{%highlight bash %}
$ cd /usr/local/etc/mail
$ vi opendkim.conf
{%endhighlight %}

Change these values in opendkim.conf:

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

Final configuration

  Generate a key to be stored in /var/db/dkim/. Consider '/usr/local/sbin/opendkim-genkey'

I'm going to use the following values for my keypair that will be used for signing outgoing
email. The public key is stuffed into the DNS using the format 'selector._domainkey.example.com IN TXT "v=DKIM1; k=rsa; p=VERYLONGSTRING"'

selector: 20231205 # a date is a common choice
domain: example.com
keylength: 2048 RSA 
privatekey_dir: /var/db/dkim/example.com/

Let's make a keypair!
{%highlight bash %}
$ opendkim-genkey -r -s 20231205 -d example.com -t 2048 -D /var/db/dkim/example.com
{%endhighlight %}

This will create two file  in `/var/db/dkim/example.com`. The private key will be **20231205.private**
and the public key will be **20231205.txt**.

Now you need to tell OpenDKIM about your private key and what domain it is associated with.

Do this by creating entries in the signing table and keytable:

{%highlight bash %}
    # cat opendkim.keytable
    20231205._domainkey.example.com example.com:20231205:/var/db/dkim/example.com/20231205.private
{%endhighlight %}

{%highlight bash %}
    # cat opendkim.signingtable
    *@example.com 20231205._domainkey.example.com
{%endhighlight %}

{%highlight bash %}
    # chgrp mailnull /usr/local/etc/mail/opendkim.*
    # chmod o-rwx /usr/local/etc/mail/opendkim.*
    # chown -R mailnull:mailnull /var/db/dkim
{%endhighlight %}

  Restart milter-opendkim service `service milter-opendkim restart`

Testing

  - Send email to `check-auth@verifier.port25.com` which will reply with a report validating DKIM, SPF, rDNS
  - Use <https://dkimvalidator.com/> to see if the key in DNS can validate signatures from from the milter
  - Check /var/log/maillog for errors
  - Check that the milter is handling incoming email correctly by looking in /var/log/maillog or mail headers of received msgs. Look for Authentication-Results header!

Operations

  How often do you publish a new DKIM key? Should I expire and publish them periodically?
 
  Publish DMARC RR like this: 

{%highlight bash %}
_dmarc.example.com TXT v=DMARC1; p=none; rua=mailto:dmarc@example.com; ruf=mailto:dmarcfail@example.com; sp=none; aspf=r; adkim=r; fo=1;
{%endhighlight %}

{%highlight bash %}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
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
 
