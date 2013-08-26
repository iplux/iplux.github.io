---
layout: post
title: "★ Blocking DNS Amplification attacks"
date: 2013-08-23 16:58
comments: true
keywords: dns, security, attack, dos, ddos, network
categories: sysadmin security
description: How to fight against DNS Amplification attacks
---

You wake up one normal day and your monitoring system shows a graph like this for the number of queries answered by your DNS server:

{% img /images/iplux/dnsreqs.png %}

Your DNS is usually used only by a couple of machines. 
What is happening?

The best way to start troubleshooting is using the command [dnstop](http://dns.measurement-factory.com/tools/dnstop/).

{% codeblock lang:bash %}
dnstop -4 eth0
{% endcodeblock %}

You will see immediately a list of the source IP that are doing the most queries to your DNS.

But what are they requesting? Just press ‘2’ and find out. You will see the domain names and the number of hits.

You have good chances that you will see one of the following domain names:

- isc.org
- ripe.net

*Congratulations*! You are victim of a [DNS amplification attack](https://www.riskanalytics.com/2013/05/23/dns-amplification-attacks/).

In a few words, the bad guys spoof their IP addresses to the one of the victim they want to attack and then they use your recursive DNS server to query domain names.

The result is that (thanks UDP!) the attackers is sending small UDP packets with the request and your DNS server is answering with DNS replies that can be many times bigger than the original packet. This replies are sent back to the spoofed IP address and your DNS is now part of a army of DOSsers to the victim.

What can you to do to avoid this problem?

You have different possibilities:

- Stop running a open recursive DNS server on Internet. There are very few reasons for having an open recursive DNS (normally only your authoritative DNS server should be available for the whole internet)

- Use the string module of iptables to block all packets that contain isc.org and ripe.

{% codeblock lang:bash %}
iptables -A INPUT -p udp -m string —hex-string “|03697363036f726700|” —algo bm —to 65535 -j DROP
{% endcodeblock %}

The command will check the content of udp packets for the hex-string that represents isc.org

- you can create some tricks on your dns server to avoid answering specific domain names. For example on dnsmasq (typical in many home routers), you could configure it with the following:

{% codeblock lang:bash %}
server=/isc.org/10.0.0.1
{% endcodeblock %}

where 10.0.0.1 is an IP that is not part of your network and does not run a dns server.

To be extra sure also null-route the IP address:

{% codeblock lang:bash %}
route add 10.0.0.1 gw 127.0.0.1 lo
{% endcodeblock %}

The result is that all requests for the domain name isc.org will be forwarded to a non-existent address and the attacker will never receive a reply.

- finally, you could also configure [fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page) to block IPs that produce a high number of DNS requests (be careful not to block valid traffic!). To do this, you can create a log message when the amounts of dns requests are higher than a certain threshold:

{% codeblock lang:bash %}
iptables -A INPUT -p udp -m udp --dport 53 -m limit --limit 5/sec -j LOG --log-prefix "fw-dns " --log-level 7
{% endcodeblock %}

This generates a log message like the following:

{% codeblock lang:bash %}
Aug 24 06:34:02 aragorn kernel: [2582889.374570] fw-dns IN=venet0 OUT= MAC= SRC=attacker.ip DST=your.ip LEN=64 TOS=0x00 PREC=0x00 TTL=110 ID=49937 PROTO=UDP SPT=43523 DPT=53 LEN=44
{% endcodeblock %}

Create the file /etc/fail2ban/filter.d/iptables-dns.conf:

{% codeblock lang:bash %}
[Definition]

failregex = fw-dns.*SRC=<HOST> DST

{% endcodeblock %}

and add the following to jail.conf to ban the IP for 1 day.

{% codeblock lang:bash %}

[iptables-dns]
enabled = true
ignoreip = 127.0.0.1 
filter = iptables-dns
action = iptables-multiport[name=iptables-dns, port="53", protocol=udp]
logpath = /var/log/iptables/dns_reqs.log
bantime = 86400
findtime = 120
maxretry = 1
{% endcodeblock %}

Do you have other suggestions to block dns amplification attacks? Alas compared to an open mail relay, sometimes there are reasons to have an open dns resolver on Internet so we will have to deal with this problem for a while.


