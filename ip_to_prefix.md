Title: How to lookup the announced ASN and prefix of an IP address
Date: Mon May 14 22:36:35 +08 2018
Category: Howto
Tags: dns, howto, bgp, whois

```
% whois -h whois.cymru.com " -p `dig +short sigsegv.run | head -n +1`"
AS      | IP               | BGP Prefix          | AS Name
13335   | 104.27.137.130   | 104.27.128.0/20     | CLOUDFLARENET - Cloudflare, Inc., US
```

The `-p` flag is needed to output the BGP prefix and the leading space and
quotes are needed to feed `whois` a single argument for the query. For full
usage check `whois -h whois.cymru.com help`

This is interesting as it just needs a `whois` client.  I didn't know whois
servers can be repurposed to serve out (general?) data. Does whois use UDP like
DNS? Is there potential for abuse (like DNS reflection using large `TXT`
records) ?

A check on tcpdump shows that whois uses TCP port 43:

```
% sudo tcpdump -pni wlp3s0 \(dst whois.cymru.com or src whois.cymru.com\) 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wlp3s0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:42:43.466149 IP 192.168.1.212.54814 > 38.229.36.132.43: Flags [S], seq 1706330394, win 29200, options [mss 1460,sackOK,TS val 2270092252 ecr 0,nop,wscale 7], length 0
22:42:43.707066 IP 38.229.36.132.43 > 192.168.1.212.54814: Flags [S.], seq 2324437127, ack 1706330395, win 28960, options [mss 1460,sackOK,TS val 1648923968 ecr 2270092252,nop,wscale 9], length 0
22:42:43.707170 IP 192.168.1.212.54814 > 38.229.36.132.43: Flags [.], ack 1, win 229, options [nop,nop,TS val 2270092493 ecr 1648923968], length 0
22:42:43.707269 IP 192.168.1.212.54814 > 38.229.36.132.43: Flags [P.], seq 1:21, ack 1, win 229, options [nop,nop,TS val 2270092493 ecr 1648923968], length 20
22:42:43.946158 IP 38.229.36.132.43 > 192.168.1.212.54814: Flags [.], ack 21, win 57, options [nop,nop,TS val 1648924209 ecr 2270092493], length 0
22:42:43.948488 IP 38.229.36.132.43 > 192.168.1.212.54814: Flags [P.], seq 1:148, ack 21, win 57, options [nop,nop,TS val 1648924212 ecr 2270092493], length 147
22:42:43.948547 IP 192.168.1.212.54814 > 38.229.36.132.43: Flags [.], ack 148, win 237, options [nop,nop,TS val 2270092734 ecr 1648924212], length 0
22:42:43.949053 IP 38.229.36.132.43 > 192.168.1.212.54814: Flags [F.], seq 148, ack 21, win 57, options [nop,nop,TS val 1648924212 ecr 2270092493], length 0
22:42:43.949144 IP 192.168.1.212.54814 > 38.229.36.132.43: Flags [F.], seq 21, ack 149, win 237, options [nop,nop,TS val 2270092735 ecr 1648924212], length 0
22:42:44.188482 IP 38.229.36.132.43 > 192.168.1.212.54814: Flags [.], ack 22, win 57, options [nop,nop,TS val 1648924451 ecr 2270092735], length 0
```
So I guess since it is not using UDP, you cannot really reflect queries to
spoofed IP addresses.

Thank you [cymru.com](http://www.team-cymru.com/) for running this public service
[Source](http://bgphints.ruud.org/articles/cymru-whois.html)
