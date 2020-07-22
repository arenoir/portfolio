---
layout: post
title:  "CARP changes in FreeBSD 10-Release"
date:   2014-02-12 11:12:34
---

**UPDATE the [FreeBSD Handbook](http://www.freebsd.org/doc/handbook/carp.html) has been updated.**

Here is a quick post regarding my struggle with CARP while upgrading a server from FreeBSD-9.2 to FreeBSD-10.

The first problem I encountered was loading the CARP kernel module. It turns out it has been renamed `carp.ko` from `if_carp.ko`.

So enable carp by adding the following line to `/boot/loader.conf`

```bash
carp_load="YES"
```

The next hangup I ran into was the carp syntax change.

You no longer define carp interfaces with `ifconfig_carp`. Instead you create a interface alias.

Here is an example of a working `rc.conf`

```bash
ifconfig_em1="inet 38.111.159.78/32"
ifconfig_em2="inet 38.111.159.78/32"

ifconfig_em1_alias0="vhid 12 advskew 210 pass RD0B4OBZ 192.168.1.11/32"
ifconfig_em1_alias1="vhid 12 advskew 210 pass RD0B4OBZ 192.168.1.12/32"

ifconfig_em2_alias0="vhid 12 advskew 210 pass RD0B4OBZ 192.168.2.11/32"
ifconfig_em2_alias1="vhid 12 advskew 210 pass RD0B4OBZ 192.168.2.12/32"

```

Note.

The alias must be numbered starting with 0 for each interface.
The alias number must be sequential, skipping numbers will result in the interface not being created.

Alternatively we could setup the interfaces using `ifconfig_<interface>_aliases` property.

```bash
ifconfig_em1_aliases="\
  vhid 12 advskew 210 pass RD0B4OBZ 192.168.1.11/32 \
  vhid 12 advskew 210 pass RD0B4OBZ 192.168.1.12/32 \
\

```

