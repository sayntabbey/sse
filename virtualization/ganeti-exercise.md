---
vim: autoread
jekyll: process
layout: default
root: ..
---

# Ganeti Exercise

## About Ganeti

[Ganeti](https://code.google.com/p/ganeti/) is a virtual machine cluster
management tool developed by Google. The solution stack uses either Xen or KVM
as the virtualization platform, LVM for disk management, and optionally DRBD
for disk replication across physical hosts.

Ganeti manages clusters of Xen or KVM host nodes. It controls and configures the
high availability features such as instance migration, DRBD disk replication and
host node failover.

You can use multiple clusters for different physical locations for example.
All the nodes in the cluster expect to have very high bandwidth between them,
for example for DRBD repication. If you have more than ~40 nodes in your
cluster, Phil Regnauld recommends that you start a new cluster rather than
adding more nodes.

## About this exercise

We will install Ganeti in some virtual machines, configure it to use the Xen
hypervisor, and use it to create and manage some virtual machines.

Normally you would install this on your physical hosts. We are using it in a
VirtualBox virtual machine which is pretending to be our physical host, because
we don't have enough physical boxes for everyone. This forces us to use Xen
(which is slower than KVM) because we can't use KVM inside a VirtualBox virtual
machine. You could use either for a real deployment. The installation process
is slightly different. KVM is not covered here.

This is not a production setup! This is intended to give you hands-on experience
of Ganeti and Xen on your own hardware (laptop) in a classroom environment. I will
try to point out where we are taking shortcuts or unusual configurations that we
would not normally do in production. NSRC has a [five-day workshop](https://nsrc.org/workshops/2015/ripe-nsrc-virt/)
on virtualisation and we have ~3 hours, so this is just an introduction!

## Install the first Node (VM host machine)

Install VirtualBox or make sure you are running version 4.3 or higher.

### Configure VirtualBox networking

Open VirtualBox Preferences > Network > Host-Only Adaptors. Ensure that you
have at least two listed: vboxnet0 and vboxnet1. If not, click on the Add
button to the right of the list to create them.

Double-click on *vboxnet0* and check that the IP addresses are as follows:

* IP address: 192.168.56.1
* Subnet mask: 255.255.255.0

And check that the DHCP server is enabled and configured for:

* Server address: 192.168.56.1
* Server mask: 255.255.255.0
* Lower address bound: 192.168.56.100
* Upper address bound: 192.168.56.200

![Enabling the DHCP server](virtualbox-disable-host-network-dhcp.png)

If you have made any changes, then **exit and restart VirtualBox**, otherwise
this change will not take effect, as we discovered after an hour of debugging!

Create a new VM called Ganeti Demo. Give it 2 GB RAM and a 40 GB VDI disk,
dynamically sized.

### Install the OS for each Node

Start the VM and attach the debian-8.x.x-amd64-DVD-1.iso image. Read the
following sections **before** you start the installation, and use them at the
appropriate times during the
installation.

#### Hostname

You must use a fully qualified hostname, for example `ganeti1.pcXX.sse.ws.afnog.org`.
If you have your own delegated DNS domain, and you know how to add A records to it,
then you can use it instead! 

We will need a cluster address which is separate from each host node's address. So
I will use names like this in the example:

* **cluster**.pcXX.sse.ws.afnog.org
* **ganeti1**.pcXX.sse.ws.afnog.org
* **ganeti2**.pcXX.sse.ws.afnog.org

You can use multiple clusters for different physical locations for example.
All the nodes in the cluster expect to have Layer 2 (shared LAN) between them
(for live migration of instances to another node) and very high bandwidth (for
DRBD replication).§ If you have more than ~40 nodes in your cluster, Phil
Regnauld recommends that you start a new cluster rather than adding more nodes.

#### Partitioning

The server should use LVM for disk space, so instead of the default *Guided
Partitioning*, choose *Manual*, then *SCSI1*. Write a new partition table if prompted.

* Select *pri/log free space*, *Create a new partition*, `max`, *Primary*,
  *Use as > Physical volume for LVM*, *Done*.
* Select *Configure the Logical Volume Manager*:
	* Select *Create volume group*
	* Enter a name for the new main volume group, for example `xenvg`. Select device `/dev/sda1`.
	* Select *Create logical volume*, enter the *Name* `Root` and *Size* `8GB`.
	* Select *Create logical volume*, enter the *Name* `Swap` and *Size* `4GB`.
	* Select *Finish*.
* Select *LVM VG xenvg, LV Root* -> *#1 8.0 GB*:
	* Select *Use as > Ext4 journalling filesystem*.
	* Select *Mount point > / (root)*.
	* Select *Done setting up the partition*.
* Select *LVM VG xenvg, LV Swap* -> *#1 4.0 GB*:
	* Select *Use as > swap area*.
	* Select *Done setting up the partition*.
* Select *Finish partitioning and write changes to disk*.

#### Proxy Server

If you are following this exercise at an AfNOG event, please enter this proxy
server name when prompted, to save a LONG install time:

* <http://196.200.223.144:3142>

Please enter this carefully and check it. Using the wrong value will make it
impossible for you to install any packages.  Of course, if you are not at the
AfNOG workshop then this server will no longer exist, so use a local proxy
server or leave it blank.

While the installation proceeds, familiarise yourself with the terminology of
[Ganeti](http://docs.ganeti.org/ganeti/2.15/html/admin.html#ganeti-terminology).

#### Software selection

Enable installation of the *SSH Server* and *Standard system utilities*,
disable everything else.

### Reconfigure the virtual network

After installation, shut down the machine and reconfigure its network interfaces in VirtualBox:

* Adapter 1: If you have a wired LAN connection, then you can set this to Bridged. Don't
  try to do that if you are using wireless and don't have a wired LAN connection!
* Adapter 2: Enabled, Host-only network, vboxnet0, set Promiscuous Mode to *Allow All*.

![Configuring Network Adaptor 1](virtualbox-configure-adaptor-2.png)

Then start the machine again. Log in on the console, edit `/etc/apt/sources.list` and delete the
`deb cdrom` line. Then install some packages:

	su
	apt install bridge-utils sudo
	usermod -G sudo afnog

Then edit `/etc/network/interfaces` to look like this:

	# The loopback network interface
	auto lo
	iface lo inet loopback

	# The primary network interface
	auto eth0
	iface eth0 inet dhcp

	auto xen-br0
	iface xen-br0 inet static
		# This must match the address of your hostname in the DNS, not the cluster address!
		address 192.168.56.11
		netmask 255.255.255.0
		bridge_ports eth1
		bridge_stp off
		bridge_fd 0


Now you should be able to reboot the host and login with SSH using the address that you assigned to
`xen-br0` above. SSH is much more convenient than the console, because you can copy and paste!

### Enable Debian Backports

Edit `/etc/apt/sources.list` and add the following line:

	deb http://ftp.debian.org/debian jessie-backports main

Then run:

	sudo apt update

### Hostnames and DNS

Edit `/etc/hostname` and put the fully-qualified hostname (FQDN) in there. Then change the
hostname to match:

	hostname `cat /etc/hostname`

Edit `/etc/hosts` and ensure that it contains the IP address and hostname of
your host. You will also need to choose a name (hostname) and IP address for
your cluster, which must be different. For example:

	127.0.0.1       localhost
	192.168.56.10   cluster.pcXX.sse.ws.afnog.org
	192.168.56.11   ganeti1.pcXX.sse.ws.afnog.org
	192.168.56.12   ganeti2.pcXX.sse.ws.afnog.org

You must use fully qualified hostnames for your nodes and for the cluster, for
example `ganeti1.pcXX.sse.ws.afnog.org`, and they must all have IP addresses.
If you have your own delegated DNS domain, and you know how to add A records to
it, then you can use it instead! 

### Configure Xen

Edit `/etc/default/grub` and add/change the following lines to enable Xen (you
would not need this for a KVM cluster in production):

	GRUB_DEFAULT=2
	GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=min:600M,max:600M"

This restricts the master domain to 600 MB RAM, which will make it slow, but give us more RAM free
for guests. In your own configurations you should probably allocate more RAM to the host (domain 0)!

Then run the following commands:

	sudo update-grub
	sudo apt dist-upgrade	
	sudo apt install xen-linux-system-amd64

Then `reboot` the host. Be sure to select a Xen kernel from the boot list. Log in again and check
that the `free` command reports 600 MB of total Mem, not 2 GB:

	afnog@ganeti:~$ free
		     total       used       free     shared    buffers     cached
	Mem:        527640

### Install DRBD

The Ganeti manual has instructions for this, but they are confusing and
out-of-date for Debian >= Wheezy, so we skip that step and do it here instead:

	sudo apt install drbd-utils

Edit `/etc/modprobe.d/drbd.conf` and make it look like this:

	options drbd minor_count=128 usermode_helper=/bin/true

Edit `/etc/modules` and add the following line at the end:

	drbd

Load the kernel module (driver) now:

	sudo modprobe drbd

### Enable Debian Backports

Edit `/etc/apt/sources.list` and add the following line:

	deb http://ftp.debian.org/debian jessie-backports main

Then run:

	sudo apt update

### Install Ganeti software

Start by running the following commands:

	sudo apt -t jessie-backports install ganeti ganeti-instance-debootstrap

Then start following the [Ganeti installation tutorial](http://docs.ganeti.org/ganeti/2.15/html/install.html),
skipping the following sections:

* Anything to do with KVM (we're using Xen instead)
* Installing th Hypervisor (we did that already)
* Installing DRBD (we already did that)
* Installing RBD: skip to
  [Installing Gluster](http://docs.ganeti.org/ganeti/2.15/html/install.html#installing-gluster) instead.
* KVM userspace access
* Configuring the network
* Configuring LVM: only follow the "Optional" step to configure LVM not to scan the DRBD devices for physical volumes. 
* Installing Ganeti: stop following at this point.

## Create a new Ganeti cluster

You should only do this on the **first node** in the cluster. Do not do this
step again unless you are setting up a new cluster!

Run the following command, ensuring that you use the correct cluster hostname:

	sudo mkdir /root/.ssh
	sudo gnt-cluster init --vg-name xenvg --enabled-hypervisors=xen-pvm -H xen-pvm:xen_cmd=xl cluster.pcXX.sse.ws.afnog.org

The `gnt-cluster` command should take a few minutes to complete.

**Note:** Normally you would use either `xen-hvm` or `kvm` as the hypervisor,
instead of `xen-pvm` above. In this case we must use `xen-pvm` because we are
doing this inside a virtual machine, so we can't use the virtualisation CPU
instructions because VirtualBox is already using them to run the Ganeti host
node (VirtualBox guest).

Test that you can run the following useful commands and examine their output:

	sudo gnt-cluster verify
	sudo gnt-node list
	sudo gnt-instance list
	sudo gnt-job list

You can ignore the following errors and warnings from `gnt-cluster verify`. The error
is out of date, and the warnings are not problems because not all LVM volumes are
used by Ganeti.

	Thu Jun  2 13:17:09 2016   - WARNING: node ganeti1.pc40.sse.ws.afnog.org: volume xenvg/Swap is unknown
	Thu Jun  2 13:17:09 2016   - WARNING: node ganeti1.pc40.sse.ws.afnog.org: volume xenvg/Root is unknown

The [Ganeti manual page](http://docs.ganeti.org/ganeti/2.15/html/man-ganeti.html) gives
useful information about Ganeti commands, including examples.

Create the file `/etc/ganeti/vnc-cluster-password` containing the password that
you want to use for VNC access to consoles.

## Test your setup

Check that the `gnt-node list` command shows your node:

	$ sudo gnt-node list
	Node                     DTotal DFree MTotal MNode MFree Pinst Sinst
	ganeti1.sse.ws.afnog.org  40.0G 28.8G   2.0G  1.9G  126M     0     0

> **Warning:** If you see question marks in all the columns after the node name, like this:
>
>	$ sudo gnt-node list
>	Node                     DTotal DFree MTotal MNode MFree Pinst Sinst
>	ganeti1.sse.ws.afnog.org      ?     ?      ?     ?     ?     0     0
>
> that means that Ganeti cannot retrieve information about your node. Check the node daemon logfile
> `/var/log/ganeti/node-daemon.log` for possible error messages. For example, if you find this error:
>
>	ERROR Can't retrieve xen hypervisor information (exited with exit code 1): ERROR:  A different toolstack (xl) have been selected!
>
> that means that Ganeti is trying to use the old `xm` command to get information, instead of the new `xl` command,
> and not getting any information. You probably forgot to add the option `-H xen-pvm:xen_cmd=xl` when you created
> the cluster. You can fix it by modifying the cluster settings on the node:
>
>	sudo gnt-cluster modify -H xen-pvm:xen_cmd=xl
>
> and check that the `gnt-node list` now shows the correct information for your node.
{: .warning}

You should also make sure that the `MFree` column shows at least 1 GB free (not
126 MB as in the example output above). This ensures that there is enough RAM
free in the hypervisor to create new guests. Otherwise you won't be able to do
much with your new hypervisor. If it doesn't show enough free RAM, check that
you have [reconfigured GRUB and run
`update-grub`](http://askubuntu.com/a/191489/49566).

Add an entry to `/etc/hosts` for a host to use for burnin testing, for example `burnin.example.com`:

	192.168.56.20   burnin.example.com

The `burnin` test will
[fail](https://groups.google.com/forum/#!topic/ganeti/ds0TwfroS8A) unless we
generate a DH parameters file for SSL:

	openssl dhparam -out dhparams.pem 2048
	cat dhparams.pem | sudo tee -a /var/lib/ganeti/server.pem

Run the `burnin` test to make sure that everything is working properly:

	sudo /usr/lib/ganeti/tools/burnin -o debootstrap+default -t plain --disk-size 1024 --mem-size=512 burnin.example.com -vv

The output should end with:

	- Attaching and removing disks
	  * instance burnin.example.com
	    attaching a disk with name RQLX441M
	    removing last disk
		Thu Jun  2 19:13:17 2016  - WARNING: Hotplug is not supported: Hotplug is not supported by this hypervisor
		Thu Jun  2 19:13:17 2016  - INFO: Modification will take place without hotplugging.
		Thu Jun  2 19:13:18 2016  - INFO: Waiting for instance burnin.example.com to sync disks
		Thu Jun  2 19:13:18 2016  - INFO: Instance burnin.example.com's disks are in sync
		Thu Jun  2 19:13:19 2016  - WARNING: Hotplug is not supported: Hotplug is not supported by this hypervisor
		Thu Jun  2 19:13:19 2016  - INFO: Modification will take place without hotplugging.
	- Removing instances
	  * instance burnin.example.com

If this fails, you may need to remove the instance before trying again:

	sudo gnt-instance remove burnin.example.com

### Create a Virtual Machine

Add a hostname (to `/etc/hosts` or to a DNS domain that you control) with a
dummy IP address. We will use `test.example.com` for this example.

	192.168.56.21	test.example.com

Normally you would give it a static IP address, but we are on
a DHCP network with limited IP addresses here.

Use the following command to create the test VM:

	sudo gnt-instance add -t plain --disk 0:size=4G -B memory=512 \
		-H xen-pvm:initrd_path=/boot/initrd-3-xenU -o debootstrap+default \
		-n ganeti.pc40.sse.ws.afnog.org test.example.com

This new VM will have the following settings:

* Disk storage as an LVM logical volume of size 4 GB.
* 512 MB memory.
* Xen paravirtualised with an initial ramdisk containing the Xen paravirt drivers
  (to give access to the root filesystem). With KVM or Xen-HVM you would not need to
  specify the `-H` parameter at all.
* Install Debian (7 in this case).
* Hosted on our first Ganeti node, `ganeti.pc40.sse.ws.afnog.org` (you must specify
  the host node or install an `iallocator` script to choose one automatically).
* With hostname and instance name `test.example.com`.

The output should look like this:

	Wed Jun  1 11:15:37 2016 * disk 0, size 4.0G
	Wed Jun  1 11:15:37 2016 * creating instance disks...
	Wed Jun  1 11:15:37 2016 adding instance test.example.com to cluster config
	Wed Jun  1 11:15:37 2016 adding disks to cluster config
	Wed Jun  1 11:15:38 2016  - INFO: Waiting for instance test.example.com to sync disks
	Wed Jun  1 11:15:38 2016  - INFO: Instance test.example.com's disks are in sync
	Wed Jun  1 11:15:38 2016  - INFO: Waiting for instance test.example.com to sync disks
	Wed Jun  1 11:15:39 2016  - INFO: Instance test.example.com's disks are in sync
	Wed Jun  1 11:15:41 2016 * running the instance OS create scripts...

If it doesn't, look for an error message in the output or in `/var/log/ganeti/jobs.log`.
Most likely one of the `gnt-instance` parameters was missing or incorrect.

Check that your new instance appears in the output of `gnt-instance list`, with status `running`:

	afnog@ganeti:/tmp$ sudo gnt-instance list
	Instance         Hypervisor OS                  Primary_node                 Status  Memory
	test.example.com xen-pvm    debootstrap+default ganeti.pc40.sse.ws.afnog.org running   512M

Connect to its console:

	sudo gnt-instance console test.example.com

You should see it boot up (if you're not too late) or a login prompt. If it's not running, you
can try to start it again and connect to the console quickly to see what happened:

	sudo gnt-instance start test.example.com
	sudo gnt-instance console test.example.com

You can exit the console by pressing `^]` (Ctrl+]).

You should be able to login as `root` with no password at the console. Add a normal user and
change the root password:

	adduser afnog
	passwd root

And edit `/etc/network/interfaces` to configure networking, making it look like this:

	auto lo
	iface lo inet loopback

	auto eth0
	iface eth0 inet dhcp

Then tell it to reboot and watch it come back up. Check that you can access the internet
from the guest. You should also be able to access the guest from your host computer, and
hopefully from the local network too, using its IP address (assigned by DHCP).

### Add a second Node

We want to create a replicating Instance (virtual machine) with high availability, so
we need a second Node to host the backup copy of this Instance.

Create a new virtual machine and configure it using the same instructions as above,
up to the point of creating a cluster, but don't create a new cluster. Use a different
hostname, for example `ganeti2.pcXX.sse.ws.afnog.org`, which must also be listed in
`/etc/hosts`. The `/etc/hosts` file should be exactly the same on both hosts.

Enable remote root logins by SSH on the new server, by editing `/etc/ssh/sshd_config`
and changing the `PermitRootLogin` setting to `yes`. Restart the `ssh` service.

Now add this node to the existing cluster, by running the following command on a node
already in the cluster (ganeti1):

	sudo gnt-node add ganeti2.pcXX.sse.ws.afnog.org

This will prompt you to login as root to the new node (ganeti2) and will forcibly add
it to the cluster. Do not do this to a node which is already in a cluster, as it will
probably result in data loss!

Verify the cluster again:

	sudo gnt-cluster verify

### Test HA with DRBD replication

We can now use the same `burnin` command with a different storage type, `drbd`, which will
create an instance that replicates its data to another node:

	sudo /usr/lib/ganeti/tools/burnin -o debootstrap+default -t drbd --disk-size 1G --mem-size=600 burnin.example.com -vv

### Install snf-image

This tool allows you to install many different distributions in a Ganeti instance, not just Debian 7.
Unfortunately there are some problems with the packaging at the moment, so we need to download and
install it manually:

	sudo apt install curl mbr python-prctl python-scapy
	sudo wget http://apt.dev.grnet.gr/jessie/snf-image_0.19-1~jessie_all.deb
	sudo dpkg -i snf-image_0.19-1~jessie_all.deb

When asked for the URL to download snf-image-helper image from, if you are following this
at an AfNOG event then you can use the copy that we downloaded for you, which will be much faster,
by entering this URL:

	http://sse-mini1.mtg.afnog.org







### Enable Remote API

Choose a username and password for your remote account (`jack` and `mypassword` in this case) and
generate a hash using `echo` and `openssl md5` like this:

	$ echo -n 'jack:Ganeti Remote API:mypassword' | openssl md5
	(stdin)= 5ede44dba4dd4e9ce3909246515b2cdc

Insert them both into `/var/lib/ganeti/rapi/user`, prefixing the password hash
with `{ha1}`, and giving this user `write` permissions:

	jack	{ha1}5ede44dba4dd4e9ce3909246515b2cdc	write

### Install the Web Manager

Run the following commands:

	sudo apt install git libssl-dev virtualenv
	cd /tmp
	wget http://sse-mini1.mtg.afnog.org/0.11.1.tar.gz
	tar xzvf 0.11.1.tar.gz
	cd ganeti_webmgr-0.11.1

We are using Ganeti Web Manager 0.11.1, which has a bug that we need to
[fix](https://github.com/osuosl/ganeti_webmgr/pull/93) before we install:

	cd ganeti_webmgr/ganeti_web/settings
	wget https://raw.githubusercontent.com/qris/ganeti_webmgr/db1712973616765f62f85c16c246b53a73e8ac4e/ganeti_webmgr/ganeti_web/settings/base.py
	cd .././..

Then we can run the installation script:

	sudo ./scripts/setup.sh

Edit the file scripts/vncauthproxy/init-systemd and if the last line contains only a `~` character,
delete it. Also, in the `[Service]` section, add the following line:

	User=www.data

Then copy it to the `systemd` service directory, and start it:

	sudo cp scripts/vncauthproxy/init-systemd /lib/systemd/system/vncauthproxy.service
	sudo mkdir /var/log/vncauthproxy /var/run/vncauthproxy
	sudo chown www-data /var/log/vncauthproxy /var/run/vncauthproxy
	sudo service vncauthproxy start
	sudo cp ganeti_webmgr/ganeti_web/settings/config.yml.dist /opt/ganeti_webmgr/config/config.yml

Edit `/opt/ganeti_webmgr/config/config.yml` and change the `EMAIL_HOST` and
`DEFAULT_FROM_EMAIL` lines, so that their values refer to your outbound server
and your email address.

Also find this line:

	NAME: /opt/ganeti_webmgr/ganeti.db

And change it to:

	NAME: /opt/ganeti_webmgr/db/ganeti.db

Then finish the installation:

	cd /opt/ganeti_webmgr
	sudo mkdir .settings db whoosh_index
	sudo chown www-data .settings db whoosh_index
	export DJANGO_SETTINGS_MODULE=ganeti_webmgr.ganeti_web.settings
	sudo -E -u www-data bin/django-admin.py syncdb --migrate
	sudo -E -u www-data bin/django-admin.py refreshcache
	sudo -E -u www-data bin/django-admin.py rebuild_index
	sudo -E -u www-data bin/django-admin.py collectstatic

Enter a username, password and email address for a super user for the Ganeti web manager.

Now start the web server in debugging mode:

	cd /opt/ganeti_webmgr
	sudo -E -u www-data bin/django-admin.py runserver 0.0.0.0:8000 --insecure

This will start the debugging webserver on port 8000, so you can check that
everything is working by visiting <http://192.168.56.10:8000>. You should get a
white page with a login and password box, but no styling (colours, images,
etc.) If not, check the console output for error messages.

Create the file `/opt/ganeti_webmgr/wsgi.py` with the following contents:

	import os
	import sys

	path = '/opt/ganeti_webmgr'

	# activate virtualenv
	activate_this = '%s/venv/bin/activate_this.py' % path
	execfile(activate_this, dict(__file__=activate_this))

	# add project to path
	if path not in sys.path:
	    sys.path.append(path)

	    # configure django environment
	    os.environ['DJANGO_SETTINGS_MODULE'] = 'ganeti_webmgr.ganeti_web.settings'

	    import django.core.handlers.wsgi
	    application = django.core.handlers.wsgi.WSGIHandler()

Create the file `/etc/apache2/sites-enabled/ganeti.conf` with the following contents:

	WSGIPythonHome /opt/ganeti_webmgr/venv
	WSGISocketPrefix /var/run/wsgi
	WSGIRestrictEmbedded On

	<VirtualHost *:80>
		ServerAdmin your-email-address@example.com
		ServerName ganeti-server.local
		ServerAlias 192.168.56.10

		# Static content needed by Django
		Alias /static "/opt/ganeti_webmgr/collected_static/"
		<Location "/static">
			Order allow,deny
			Allow from all
			SetHandler None
		</Location>

		# Django settings - AFTER the static media stuff
		WSGIScriptAlias / /opt/ganeti_webmgr/wsgi.py
		WSGIDaemonProcess ganeti processes=1 threads=10 display-name='%{GROUP}' deadlock-timeout=30
		WSGIApplicationGroup %{GLOBAL}
		WSGIProcessGroup ganeti

		# Possible values include: debug, info, notice, warn, error, crit,
		# alert, emerg.
		LogLevel warn

		<Location />
			Require all granted
		</Location>
	</VirtualHost>

Now you should be able to access http://192.168.56.10/ (without the :8000 port specification)
and see the login page with graphics:

![Ganeti Login Screen](ganeti-login.png)

Log in using the superuser account that you created during the `syncdb`
command, or if you have forgotten the details, run this command to create a new
one:

	sudo -u www-data venv/bin/python manage.py createsuperuser

Choose *Clusters* from the menu on the left, and then click *Add Cluster* in the top right.
Enter the following details:

* Hostname: localhost
* Port: 5080
* Description: My Cluster

Leave the other details blank, and click *Add*. Your new cluster should then
appear with its specifications:

![Ganeti Manager showing the new cluster](ganeti-my-cluster.png)
