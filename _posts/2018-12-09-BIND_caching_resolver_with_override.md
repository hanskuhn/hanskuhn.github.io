---
layout: post
title: BIND caching resolver with internal RRs
published: false
---

I needed to create a caching forwarder on a campus in Tanzania and 
they host their DNS on a cloud provider. They had some campus resources
that were only accessible on their LAN. For those hosts they wanted to
return a local IP address, rather than forward their query to their 
REN.

{%highlight bash %}
; file: /etc/bind/db.saut.ac.tz
@                      3600 SOA   localhost.  root.localhost. (
                              2018120901                 ; serial number
                              3600                       ; refresh
                              600                        ; retry
                              604800                     ; expire
                              1800                     ) ; minimum ttl
			NS	localhost.

www.opac.mlrc.saut.ac.tz	A	192.168.1.2
www.ebooks.saut.ac.tz		A	192.168.1.4
opac.mlrc.saut.ac.tz		A	192.168.1.5
teeal.saut.ac.tz		A	192.168.1.6

;www			CNAME	saut.ac.tz.
{%endhighlight %}

{%highlight bash %}
// file: /etc/bind/named.conf.local
zone "saut.ac.tz" {
  type master;
  file "/etc/bind/db.saut.ac.tz";
};
{%endhighlight %}

{%highlight bash %}
// file: /etc/bind/named.conf.options

acl sautusers {
        192.168.0.0/16;
        localhost;
        localnets;
};

options {

	directory "/var/cache/bind";

	response-policy { zone "saut.ac.tz"; };

	recursion yes;
        allow-query { sautusers; };
//        auth-nxdomain no;    # conform to RFC1035


	forwarders {
#	 	41.93.50.3;
#		41.93.50.4;
8.8.8.8;
	};

	//========================================================================
	// If BIND logs error messages about the root key being expired,
	// you will need to update your keys.  See https://www.isc.org/bind-keys
	//========================================================================
	dnssec-validation auto;

	auth-nxdomain no;    # conform to RFC1035
	listen-on-v6 { any; };
};

logging {
  channel rpz-queries {
    file "/var/log/bind/saut.ac.tz_internal.log" versions 10 size 500k;
    severity info;
  };
  category rpz {
    rpz-queries;
  };
};
{%endhighlight %}

I had to modify /etc/apparmor.d/usr.sbin.named to allow bind to write to the logfile.

{%highlight bash %}
/var/log/bind/saut.ac.tz_internal.log rw,
{%endhighlight %}
