---
vim: autoread
jekyll: process
layout: default
root: ../..
---

# VM admin howto

The following documents might be useful additional references:

* [Server setup](../server/index.html) for the host (sse-nuc1.mtg.afnog.org?)
* [LXC setup](../lxc/index.html) for running ~40 containers on the host
* [Guest setup](../guest/index.html) for each container

## Host access

SSH to inst@sse-nuc1.mtg.afnog.org (once DNS is set up) or inst@196.200.223.144.
Password is the usual, adapted to this year. Email me if you don't know it.

Once on the host, the following commands may be useful:

* `lxc-ls --fancy` to list all the containers, pc1 through pc40
* `lxc-attach --name pcX` to attach to a container (virtual console)
* `lxc-start --name pcX` to start a container
* `lxc-stop --name pcX` to stop a container
* `lxc-stop --name pcX -r/--reboot` to reboot a container
* `lxc-stop --name pcX -k/--kill` to kill (force shutdown) a container
* `lxc-destroy --name pcX` to completely delete a container (irreversibly)
* `lxc-copy --name pcX --newname pcY` to clone an existing container with a new name
* `lxc-autostart` to start all the containers
* `lxc-autostart -s/--shutdown` to shutdown all the containers cleanly
* `lxc-autostart -r/--reboot` to reboot all the containers cleanly
* `lxc-autostart -k/--kill` to kill (force shutdown) all the containers

I've assigned hostnames to each container by editing `/etc/hostname`, and IP addresses by editing `/etc/network/interfaces`.
You'll need to redo that if you destroy and re-clone a container (otherwise you'll have an IP address conflict).

The guests all have IP addresses in the 196.200.219.101-140 range, where pcX = 196.200.219.(X + 100).
External routing for the 196.200.219.0/24 subnet is not available at this time, hopefully it will be soon.