Title: How to search for packages in OpenBSD
Date: Sat Apr 22 17:06:30 +08 2017
Category: Howto
Tags: openbsd, howto

I just can't seem to find any good documentation on any utility to search and
discover OpenBSD packages (similar to `apt-file search` or `yum whatprovides`
perhaps). What we can do is to list out all the available tarballs and
hopefully divine something based on the filenames.

`curl -L $(cat /etc/installurl)/$(uname -r)/packages/$(uname -m) | grep -oE '[^>" ]+\.tgz'`

Update Fri Jun 30 15:32:56 +08 2017: There is a built in utility. It is  `pkg_locate`
