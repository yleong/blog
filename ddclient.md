Title: How to use and configure ddclient
Date: Mon Apr 10 21:39:46 SGT 2017
Category: Howto
Tags: dns, howto

There are 2 parts to configure in `/etc/ddclient/ddclient.conf`

1. Configure a method to retrieve the current WAN IP address
1. Configure a method to update your DNS record

For the first, if your router is supported, you can just uncomment the relevant
lines for your model.. Otherwise, you can go for either `web` or `cmd` methods.
For `web` method, you just provide some URL that returns the IP as a web page,
e.g.  `web=dynamicdns.park-your-domain.com/getip` and ddclient will perform a
HTTP GET to retrieve the IP. For `cmd` method, you just provide some shell
command that returns the desired IP on STDOUT.

For the second part, you can check if your domain provider is already listed in
the sample `ddclient.conf`. Each domain provider has their own protocol, but
there should at least be a web address, a login and a password so that ddclient
is able to login to your provider and update your DNS records.
