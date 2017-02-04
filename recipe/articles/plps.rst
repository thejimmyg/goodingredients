Protocol-Level Proxies
++++++++++++++++++++++

.. contents ::

Now that OpenVZ is running we can begin work on the protocol level proxies. For
background information on what the PLPs are and why they are useful read the
`Architecture <architecture.html>`_ document.

Removing Unnecessary VEs
========================

If you followed the `OpenVZ tutorial <set-up-openvz.html>`_ you have already
set up a VE named ``2`` which you can use as a Protocol-Level Proxy (PLP). If
you want to start again though you can delete  VE ``2`` like this:

::

    james@mail:~$ sudo vzctl stop 2
    Stopping VE ...
    VE was stopped
    VE is unmounted
    james@mail:~$ sudo vzctl destroy 2
    Destroying VE private area: /var/lib/vz/private/2
    VE private area was destroyed

Setting the Mountpoint
======================

The configuration files for the VEs are stored in ``/etc/vz/conf/2.conf`` and
``/etc/vz/conf/3.conf``. The VEs themselves are stored in
``/var/lib/vz/private/``:

::

    $ ls -l /var/lib/vz/private/
    total 8
    drwxr-xr-x 20 root root 4096 2008-12-24 17:09 2
    drwxr-xr-x 20 root root 4096 2008-12-24 17:09 3

They take about 160Mb each:

::

    $ sudo du -hs /var/lib/vz/private/
    314M	/var/lib/vz/private/

Over time though this could grow significantly so if you are using the
`Production Setup <production-debian-lenny-install.rst>`_ or the `KVM Virtual
Production Installation <kvm-virtual-production-installation.rst>`_ for the HE,
it is worth having them on their own logical volume.

.. tip ::

    It is a good idea to name the virtual machines according to the last
    numbers in their IP addresses so you don't get confused. In this recipe the
    virtual machines are names 2 and 3 but you will need to use your own numbers if
    they are different.

Since the PLPs won't have any user data on them they can actually be quite
small. To be on the safe side though we'll give the 10Gb space each. We can
always increase this later using LVM.

Let's create a logical volume for each of the servers:

::

    $ sudo lvcreate -L 10G -nve2 vg_data
      Logical volume "ve2" created
    $ sudo lvcreate -L 10G -nve3 vg_data
      Logical volume "ve3" created

Here ``ve2`` and ``ve3`` are the names of the two logical volumes, 10G is the
size of each and ``vg_data`` is the name of the volume group to create them in.
The logical volume size can be bigger than the size of the underlying
partitions (LVM works out how to store the data correctly) but it can't be
bigger than the size of the free space in the volume group.

You'll see the two new devices have been created:

::

    $ ls /dev/vg_data/
    ve2  ve3
    $ sudo lvscan
      ACTIVE            '/dev/vg_data/ve2' [10.00 GB] inherit
      ACTIVE            '/dev/vg_data/ve3' [10.00 GB] inherit
      ACTIVE            '/dev/vg_root/root' [20.00 GB] inherit
      ACTIVE            '/dev/vg_root/swap' [8.00 GB] inherit
      ACTIVE            '/dev/vg_root/tmp' [2.00 GB] inherit

Now we need to create the filesystems:

::

    $ sudo mkfs -t ext3 /dev/vg_data/ve2
    $ sudo mkfs -t ext3 /dev/vg_data/ve3

At this point we are going to do something a bit clever. We'd like to use a
different logical volume for each VE but OpenVZ won't create the VE in a folder
that already exists and if we don't mount the logical volumes we've just
created we won't be able to put anything in them.

Instead create a directory ``/var/lib/vz/lv`` for the logical volumes and mount them there:

::

    $ sudo mkdir /var/lib/vz/lv

Now we'll create another config for our logical volume setup:

::

    $ sudo cp /etc/vz/conf/ve-vps.basic.conf-sample /etc/vz/conf/ve-vps.lv.conf-sample


Edit the new file ``/etc/vz/conf/ve-vps.lv.conf-sample`` and add this line to the end:

::

    VE_PRIVATE="/var/lib/vz/lv/$VEID/ve"

Now we can use the ``vps.lv`` config to create VEs which automatically use a mountpoint in ``/var/lib/vz/lv``. 

Set the mountpoint of ``/var/lib/vz/lv/2`` to ``/dev/vg_data/ve2`` and
that of ``/var/lib/vz/lv/3`` to ``/dev/vg_data/ve3``. Add the following to
the end of ``/etc/fstab``:

::

    /dev/vg_data/ve2  /var/lib/vz/lv/2  ext3  defaults 0 0
    /dev/vg_data/ve3  /var/lib/vz/lv/3  ext3  defaults 0 0

Then create the directories we need:

::

    $ sudo mkdir /var/lib/vz/lv/2 
    $ sudo mkdir /var/lib/vz/lv/3 

Now you can mount the directories:

::

    $ sudo mount /var/lib/vz/lv/2
    $ sudo mount /var/lib/vz/lv/3

and you are ready to go!

.. tip ::

    If you make a mistake creating a logical volume you can remove it with this
    command but make sure you specify the correct logical volume or you could lose
    all the data on a logical volume you wanted to keep!

    ::

        $ sudo lvremove /dev/vg_data/ve3

At this point it is worth rebooting to check that the directories are correctly
mounted from the data in ``/etc/fstab``.

Creating the VEs
================

Now create the two virtual machines, each must have a unique IP address. The
first will have the hostname ``plpa.example.com`` and IP address
``192.168.100.2`` and the second will use ``plpb.example.com`` and have the IP
address ``192.168.100.3``. You should replace ``example.com`` with your own
domain and use your own IP addresses. Also you'll need to replace the
nameserver values uses with your own. You can use multiple ``--nameserver``
options to set more than one nameserver. If you don't know what your
nameservers are you can look in ``/etc/resolv.conf`` on the HE. We want them
both to start when the HE starts so we also set ``--onboot yes``.

Here are the commands to run on the HE (they are explained in the OpenVZ guide):

::

    $ sudo vzctl create 2 --ostemplate debian-5.0-amd64-minimal --config vps.lv
    $ sudo vzctl set 2 --hostname plpa.example.com --diskspace 8G:9G --ipadd 192.168.100.2 --nameserver 213.133.98.98 --nameserver 213.133.98.99 --nameserver 213.133.100.100 --numothersock 250 --privvmpages=256000 --onboot yes --save
    $ sudo vzctl create 3 --ostemplate debian-5.0-amd64-minimal --config vps.lv
    $ sudo vzctl set 3 --hostname plpb.example.com --diskspace 8G:9G --ipadd 192.168.100.3 --nameserver 213.133.98.98 --nameserver 213.133.98.99 --nameserver 213.133.100.100 --numothersock 250 --privvmpages=256000 --onboot yes --save

Notice that we are using the ``vps.lv`` config we just created so that the data for the virtual machines is stored on a LVM2 logical volume. 

.. tip ::

    Sometimes you might want to have the VEs in a private network without
    externally-accessible IP addresses. For example, you might like to have a
    private network 192.168.100.xxx and give one of the VEs the IP address
    192.168.100.2. In which case you'll need to run this on the HE to allow that VE
    to access the internet. The ``188.40.40.131`` needs to be replaced with the
    external IP address of the HE:

    ::

        $ sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j SNAT --to 188.40.40.131

Just for compatibility with any other software which might be expecting the
data to be in the ``/var/lib/vz/private`` directory it is best to create some
symlinks:

::

    $ sudo ln -s /var/lib/vz/lv/2/ve /var/lib/vz/private/2
    $ sudo ln -s /var/lib/vz/lv/3/ve /var/lib/vz/private/3

So that the VE is easy to identify in backups I also create a file in the directory of each one (of course you can find out from looking ``/etc/network/interfaces`` to see which IP address it is using but this seems nicer to me:

::

    $ sudo touch /var/lib/vz/lv/2/2
    $ sudo touch /var/lib/vz/lv/3/3

Now start both VEs:

::

    $ sudo vzctl start 2 
    $ sudo vzctl start 3

You can now enter the PLP as root with this command:

::

    $ sudo vzctl enter 2

.. tip ::

   There are a few things in this setup worth explaining for the interested reader:

   1. The reason for having the VE in a ``ve`` directory within the logical
      volume is that the filesystem wants to maintain a ``lost+found`` directory
      which should be owned by the HE, not the VE. 
   2. Because the VE data can't go directly in the root of a logical volume we
      can't store the data in the ``private`` directory. If we did it would actually
      be in ``2/ve``, ``3/ve`` etc because of point 1 above that would make it
      impossible to maintain an appropriate symlink becuase the directory name would
      be the same as the symlink. 

Configuring the VEs
===================

Once you have root access you should now follow each of the configuration
tutorials to configure each VE:

1. `Update and Upgrade <configure/update-and-upgrade.html>`_
2. `Add a User and Configure sudo <configure/add-a-user-and-configure-sudo.html>`_
3. `Basic SSH Security <configure/basic-ssh-security.html>`_ 
4. `Locales <configure/locales.html>`_
5. `Times and Dates <configure/time-and-date.html>`_
6. `Firewall <configure/firewall.html>`_

Once these steps are completed you can sign in directly via SSH to the PLP
using the username of the user you created in step 2 to continue the article.
For example:

::

    $ ssh owner@plpa.example.com

Setting a root Password
=======================

On each machine, set a root password:

::

    james@plpa:~ sudo -i
    root@plpa:~# passwd
    Enter new UNIX password: 
    Retype new UNIX password: 
    passwd: password updated successfully
    root@plpa:~# exit
    logout
    james@plpa:~$ 

Setting up Nginx
================

Nginx is a slightly awkward piece of software in that it is designed to be
extremely fast so modules it uses have to be statically compiled into it. Since
we want SSL support as well as other modules we need to install it from source
code. Here's how.

Install the ``dpkg-dev`` package:

::

    $ sudo apt-get install build-essential dpkg-dev

Now get the latest debian package

::

    $ sudo apt-get source nginx

Now you can edit ``nginx-0.6.32/debian/rules`` to change how the package should
be compiled. Add any extra options after the existing configure options such as
``--with-http_flv_module --with-http_ssl_module --with-http_dav_module``.

When you have made your changes, update the ``nginx-0.6.32/debian/changelog`` file:

::

    nginx (0.6.32-3) unstable; urgency=low
    
      * debian/control: added some custom flags for my own use
    
     -- James Gardner <james@example.com>  Thu, 11 June 2009 17:14:27 +0200

You have to stick to exactly this format for the changelog otherwise the
package builder will complain.

Get the build dependencies you need:

::

    $ sudo apt-get build-dep nginx

Now you can build the modified package:

::

    $ cd nginx-0.6.32/
    $ sudo dpkg-buildpackage -b -uc

The ``-uc`` in the above command means "unsigned changes". If you exclude it
you will probably get an error about signing:

::

    $ sudo dpkg-buildpackage: warning: Failed to sign .changes file

The final ``nginx_0.6.32-3_amd64.deb`` file will be in the directory above. You
can then install Nginx like this:

::

    $ sudo dpkg -i ../nginx_0.6.32-3_amd64.deb

If you use the same ``.deb`` on another machine you'll need to do this:

::

    $ sudo apt-get install libpcre3
    $ dpkg -i nginx_0.6.32-3_amd64.deb

WARNING: I don't know what will happen if there is a security update. Will this
package be overwritten with a standard one?

Configuration
=============

We want Nginx to proxy requests to another server running Apache. Let's assume
that server is on 192.168.100.2. Replace the file
``/etc/nginx/sites-enabled/default`` with this:

::

    server {
        listen   80;
        
        access_log  /var/log/nginx/localhost.access.log;
        
        location / {
            access_log off;
            proxy_pass http://192.168.100.2:80;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

Then restart Nginx:

::

    $ sudo /etc/init.d/nginx restart

Check Nginx is running:

::

    $ sudo netstat -tap 
    Active Internet connections (servers and established)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 *:www                   *:*                     LISTEN      14091/nginx     
    tcp        0      0 *:ssh                   *:*                     LISTEN      2759/sshd       
    tcp6       0      0 [::]:ssh                [::]:*                  LISTEN      2759/sshd    

As you can see, Nginx is listening on the ``www`` port.

Installing Apache
=================

We'll install Apache into a separate OpenVZ container using a private network
so that the running server can only be accessed via Nginx. First create the new
VE:

::

    $ sudo vzctl create 2 --ostemplate debian-5.0-amd64-minimal --config vps.lv
    $ sudo vzctl set 2 --hostname server1.example.com --ipadd 192.168.100.2 --numothersock 250 --privvmpages=128536 --onboot yes --save --nameserver 213.133.98.98 --nameserver 213.133.99.99 --nameserver 213.133.100.100
    
Then start it and enter it:

::

    $ sudo vzctl start 2
    $ sudo vzctl enter 2

Then confgigure it as you have done with the other servers.

Now you can install Apache:

::

    $ sudo apt-get install libapache2-mod-wsgi

If you visit the external IP of the server running Nginx, you should see the
default "It works!" message from Apache. 

You can now go into more detail and set up various types of Apache sites:

* `Set up Mercurial <mercurial.html>`_
* `Set up Apache Sites <apache.html>`_
* `Set up a Contact Form <contact_form.html>`_

Setting up Email
================

We're going to set up this system:

::

     Mail Gateways           User Servers

      +--------+
      | SMTP A | -->   +------------------------+
      +--------+       |     User group 1:      |
                       |  Spam and Virus Filter |
                       |    SMTP Relay Mail     |
      +--------+       | Dovecot IMAP and POP3  |
      | SMTP B | -->   +------------------------+
      +--------+        
  

There are going to be two mail gateways installed, one on PLP A and one on PLP
B. These servers do nothing except accept mail from the internet and forward it
on to the correct *user server* or redirect it elsewhere (to another email
address or to a different SMTP server). One mail gateway can serve as the MX
backup for the other so that mail can be delivered as long as one of the
servers is up.

The setup also requires a set of *user servers*, each of which contains an
authenticated SMTP relay, spam and virus filters and IMAP and POP3 servers.
Effectively a user interacts only with the user server their mailbox is on but
email is delivered via the two mail gateways (although it doesn't matter if
mail is delivered directly to a user server either).

The idea is that if we get too much mail for one user server to handle we can just
set up another user server and configure the mail gateways to send the mail to
the correct server internally. If we get too much mail for the mail gateways to
handle we simply add more of them and use DNS to load balance requests amongst
them. This way we should have a fairly scalable and resilient mail architecture.

.. note ::

   Of course, there is no resilience on the individual user servers. If one of
   those goes down, the users it serves won't be able to access their mail. If
   this is a big problem you could investigate DRBD as a way to duplicate user
   data across two servers but this is quite complicated. No mail is lost even
   if a user server goes down, the mail gateways should just hold onto it until
   it is back up.

You might be wondering why the spam and virus checking is done on the user
servers and not on the mail gateway. The reasons are:

* Different user groups might want different spam settings
* Spam and virus checking is farily CPU intensive and since the easiest bit 
  of the architecture to scale is the user servers, it makes sense to handle
  the CPU intensive parts there
* I've had problems with the code which updates the spam and virus database
  using up the entire CPU power for hours at a time. Although this was a bug I'd
  rather have it occur on the user servers than the gateways.

You could configure spam and virus checking on the mail gateways though if you
prefer. The instructions are the same.

Install the First Mail Gateway
------------------------------

To get started set up the first of the two mail gateways by following this tutorial:

* `Postfix Gateway <email3/postfix-gateway.html>`_

Create a User Server
--------------------

Now that the gateway is set up, let's set up a user server.

* `Postfix and Dovecot <email3/postfix-with-dovecot.html>`_

After the set up, users's will be able to login to the user server via IMAP,
IMAPS, POP3 or POP3S to retrieve their mail. They will also be able to send
mail via the same server's authenticated SMTP relay.

Set up Spam and Virus Filtering
-------------------------------

Once Dovecot is working you might like to set up spam and virus protection. You
can do so by following this tutorial:

* `Spam and Virus Protection <email3/spam-and-virus-protection.html>`_

Postfix on PLP B
----------------

If you were using a single server set up you might add resilience by setting up
an MX backup on a separate server to store all emails for the domains you
handle, and forwarding them on once the main server is back online. There is an
article describing how to do this at http://www.howtoforge.com/postfix_backup_mx.

We can do one better though. Our MX Backup can be identical to the main plpa
server and can forward mail straight onto the user server, Mail A. That
way, even if PLPA goes down, the users will still receive mail. In our setup
the database is running on PLPA so if it goes down, access to the database goes
too. This means we also need a copy of the database on PLPB and we need to tell
all three servers that they can access the copy in the event of a failure. Our
new server will be called PLPB.

Of course, if the maila server goes down the users won't be able to access
their email but both PLPA and PLPB will store it for them until they can.

Now repeat the work of creating a posfix relay by following this guide again
for PLP B with a few changes...

* `Postfix Gateway <email3/postfix-gateway.html>`_

Then simply take a dump of the data on PLPA and put it on PLPB so that both have copies.

.. warning::
   
   Be careful to ensure both servers alwyas have *exactly* the same data and that changes you make to one are duplicated on the other, otherwise you might find hard to debug intermittant problems.

Checking /etc/hosts
-------------------

In order for everything to work smoothly and to avoid slow DNS lookups it is a
good idea to explicitly put the hostnames and IP address of all the servers in
the ``/etc/hosts`` file on each of the three machines.

Ensure they have the following entires:

::

    192.168.100.2 plpa.example.com
    192.168.100.3 plpb.example.com
    192.168.100.4 maila.example.com

Setting up Nameservers
======================

In order to be in control of your own DNS settings you need to set up a nameserver.

Primary Nameserver
------------------

Follow the article below to set up a primary nameserver on PLP A:

* `Bind Primary Nameserver <bind-primary-nameserver.html>`_

Caching Nameserver
------------------

Once the primary nameserver is set up you might decide to use it as the main
nameserver for your internal network. This is particularly useful if you want
to prototype a complex network of servers on a virtual machine. 

.. caution ::

   You wouldn't want to do this is your nameservers were live on the internet for two reasons:

   1. You wouldn't be using internal network IP addresses so every IP would resolve correctly anyway
   2. You wouldn't want to respond to DNS entries that weren't for your domains which is what this configuration lets you do

   It useful for testing purposes on an internal network though.

To set up BIND as a caching nameserver too you just need to specify the real
nameservers in a ``forwarders`` section in ``/etc/bind/named.conf.options``.
Add this with your ISP's nameserver. You can add multiple IPs each with a ``;``
after them too:

::

    options {
        ...
        forwarders {
            192.168.100.1;
        };
    };

Now external lookups work too:

::

    $ ping google.com
    PING google.com (74.125.45.100) 56(84) bytes of data.
    64 bytes from google.com (74.125.45.100): icmp_seq=1 ttl=44 time=115 ms
    64 bytes from google.com (74.125.45.100): icmp_seq=2 ttl=44 time=117 ms
    64 bytes from google.com (74.125.45.100): icmp_seq=3 ttl=44 time=116 ms
    ^C
    --- google.com ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 2001ms
    rtt min/avg/max/mdev = 115.541/116.459/117.654/0.968 ms

Now it's working you can set the plpa and maila to use the same nameserver. You
can also remove the manual entries you added to ``/etc/hosts`` referencing the
other machines because they aren't needed after you've set up everything to use
the new nameserver.

Secondary Nameserver
--------------------

To provide some resilience in case the primary nameserver fails you can set up
a secondary nameserver. Set up a secondary nameserver on PLP B and have it
transfer the zones from PLP A by following this tutorial:

* `Bind Secondary Nameserver <bind-secondary-nameserver.html>`_

Upading ``/etc/resolv.conf`` Again
----------------------------------

If you are already using the primary nameserver in ``/etc/resolv.conf`` it
makes sense to also use the secondary. Modify the secondary to add the
``forwarders`` option to ``/etc/bind/named.conf.options`` just like you did for
the primary and then restart the seconday.

Now for every server which is using the primary in ``/etc/resolv.conf``, edit
it to add the secondary:

::

    nameserver 192.168.100.2
    nameserver 192.168.100.3

Now if either nameserver should fail, DNS resolution will still be possible via
the other nameserver.

