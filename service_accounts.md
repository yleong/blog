Title: Do not invoke services using bare doas
Date: Thu Apr 13 23:56:07 +08 2017
Category: OpenBSD
Tags: doas, security

OpenBSD usually run services using a service account, e.g., for `ddclient`
there is a `_ddclient` user. During testing I ran a bare `doas ddclient`
that created cache files as owner root. After that `ddclient` is no longer
able to access its cache when it runs as a service under `_ddclient`. 

So the lesson here is to test services as their service user, e.g., `doas -u _ddclient ddclient`
