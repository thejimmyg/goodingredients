Set up OpenVZ
+++++++++++++

:Pre-Requisites: Debian Installed
:Required Reading: Rsync, SSH Keys

In this tutorial you'll learn how to set up OpenVZ on a Debian 5.0 server. I'll assume you have a running server properly configured and ready to go. 

Installation
============

An OpenVZ kernel and the ``vzctl`` and ``vzquota`` packages are available in
the Debian Lenny repositories, so we can install them as follows:

::

    $ sudo apt-get install linux-image-openvz-amd64 vzctl vzquota 

This uses an additional 83 Mb space taking the total for a minimum Debian and OpenVZ to 648Mb. (Handy to know when planning partitioning schemes). 

Updating Host Settings
======================

Now uncomment the following lines in ``/etc/sysctl.conf``:

::

    net.ipv4.conf.all.rp_filter=1
    net.ipv4.ip_forward=1
    net.ipv4.icmp_echo_ignore_broadcasts=1

Then add the following at the end:

::

    net.ipv4.conf.default.forwarding=1
    net.ipv4.conf.default.proxy_arp = 0
    kernel.sysrq = 1
    net.ipv4.conf.default.send_redirects = 1
    net.ipv4.conf.all.send_redirects = 0
    net.ipv4.conf.eth0.proxy_arp=1

Now update the configuration: 

::

    $ sudo sysctl -p 

Open ``/etc/vz/vz.conf`` and set ``NEIGHBOUR_DEVS`` to all:

::

    $ sudo vim /etc/vz/vz.conf


Once installation is complete reboot into new kernel:

::

    $ sudo reboot

.. tip ::

    If you are running a virtual machine under KVM, the first time you reboot
    after installing the OpenVZ kernel on the virtual machine, it hangs. Just
    destroy the running instance and start it up again and everything works as
    expected.

First Boot of the Host Environment
==================================

In OpenVZ terminology, the host is called the *host environments* (HEs).

Once the system has booted check the kernel:

::

    $ uname -r
    2.6.26-2-openvz-amd64

Then check that OpenVZ kernel facility ``vzmond`` is running:

::

    $ ps aux | grep vz
    root      1804  0.0  0.0      0     0 ?        S    17:04   0:00 [vzmond]


Finally check a network interface for containers is present (``venet0``):

::

    $ /sbin/ifconfig
    eth0      Link encap:Ethernet  HWaddr 00:16:36:11:de:2e  
              inet addr:192.168.100.141  Bcast:192.168.100.255  Mask:255.255.255.0
              inet6 addr: fe80::216:36ff:fe11:de2e/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:1332 errors:0 dropped:0 overruns:0 frame:0
              TX packets:861 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:284375 (277.7 KiB)  TX bytes:112752 (110.1 KiB)
              Interrupt:10 Base address:0x4000 
    
    lo        Link encap:Local Loopback  
              inet addr:127.0.0.1  Mask:255.0.0.0
              inet6 addr: ::1/128 Scope:Host
              UP LOOPBACK RUNNING  MTU:16436  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0 
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
    
    venet0    Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
              UP BROADCAST POINTOPOINT RUNNING NOARP  MTU:1500  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0 
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

If everything is present we can continue.

Create a Virtual Environment
============================

In OpenVZ terminology, guests are called *virtual environments* (VEs).

Now get an image template for the VE:

::

    $ cd /var/lib/vz/template/cache
    $ sudo wget http://download.openvz.org/template/precreated/contrib/debian-5.0-amd64-minimal.tar.gz
    --2009-04-18 23:36:09--  http://download.openvz.org/template/precreated/contrib/debian-5.0-amd64-minimal.tar.gz
    Resolving download.openvz.org... 64.131.90.11
    Connecting to download.openvz.org|64.131.90.11|:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 61459687 (59M) [application/x-gzip]
    Saving to: `debian-5.0-amd64-minimal.tar.gz'
    
    53% [===================>                   ] 33,037,136   718K/s  eta 44s     

It is 59Mb:

::

    $ ls -lah debian-5.0-amd64-minimal.tar.gz 
    -rw-r--r-- 1 james james 59M 2009-01-13 07:44 debian-5.0-amd64-minimal.tar.gz

and the version I used has the following checksum:

::

    $ md5sum debian-5.0-amd64-minimal.tar.gz 
    17049c3bcc694a84975dcf12f79aa597  debian-5.0-amd64-minimal.tar.gz

Create a symlink from ``/var/lib/vz`` to ``/vz`` to provide backward compatibility:

::

    $ sudo ln -s /var/lib/vz /vz 

To set up a VE from the template you've just downloaded run:

::

    $ sudo vzctl create 221 --ostemplate debian-5.0-amd64-minimal --config vps.basic
    Creating VE private area (debian-5.0-amd64-minimal)
    Performing postcreate actions
    VE private area was created

The 221 must be a unique ID. You can use the last part of the virtual machine's
IP address for it. For example, if the virtual machine's IP address is
192.168.100.221, you can use 221 as the ID.

Set up networking:

::
 
    $ sudo vzctl set 221 --hostname test.example.com --save
    Saved parameters for VE 221
    $ sudo vzctl set 221 --ipadd 192.168.100.2 --save
    Saved parameters for VE 221

The nameservers will probably the same as those on the HE. On the host run:

::

    $ cat /etc/resolv.conf
    nameserver 192.168.100.1

Then use these IP addresses for the nameservers of the VEs. In this case ``192.168.100.1`` is the nameserver:

Set up the nameservers:

::

    $ sudo vzctl set 221 --nameserver 192.168.100.1 --save
    Saved parameters for VE 221

Set this option:

::

    $ sudo vzctl set 221 --numothersock 120 --save
    Saved parameters for VE 221

You can get information about the ``vzctl`` command with:

::

    # man vzctl

For example the ``--nameserver`` options is explained like this:

::

    --nameserver addr
        Sets DNS server IP address for a VE. If you want to set several nameservers, you should do it at once, so  use  --nameserver  option  multiple
        times in one call to vzctl, as all the name server values set in previous calls to vzctl gets overwritten.

Now start the server:

::

    $ sudo vzctl start 221 
    Starting VE ...
    VE is mounted
    Adding IP address(es): 192.168.100.221
    Setting CPU units: 1000
    Configure meminfo: 65536
    Set hostname: test.example.com
    File resolv.conf was modified
    VE start in progress...

Enter the server with a root prompt:

::

    $ sudo vzctl enter 221
    root@test:/# 

You should now be able to ping addresses on the internet, the HE and other machines connected to the network:

::

    root@test:/# ping google.com
    PING google.com (74.125.67.100) 56(84) bytes of data.
    64 bytes from google.com (74.125.67.100): icmp_seq=2 ttl=52 time=120 ms
    64 bytes from google.com (74.125.67.100): icmp_seq=3 ttl=52 time=116 ms
    ^C
    --- google.com ping statistics ---
    3 packets transmitted, 2 received, 33% packet loss, time 2018ms
    rtt min/avg/max/mdev = 116.485/118.490/120.495/2.005 ms
    root@test:/# ping 192.168.100.1
    PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
    64 bytes from 192.168.100.1: icmp_seq=1 ttl=63 time=0.788 ms
    64 bytes from 192.168.100.1: icmp_seq=2 ttl=63 time=0.590 ms
    ^C
    --- 192.168.100.1 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1002ms
    rtt min/avg/max/mdev = 0.590/0.689/0.788/0.099 ms
    root@test:/# ping 192.168.1.1
    PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
    64 bytes from 192.168.1.1: icmp_seq=1 ttl=253 time=21.6 ms
    64 bytes from 192.168.1.1: icmp_seq=2 ttl=253 time=145 ms
    ^C
    --- 192.168.1.1 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1006ms
    rtt min/avg/max/mdev = 21.696/83.472/145.248/61.776 ms

If you want to have the vm started at boot, run

::

    $ sudo vzctl set 221 --onboot yes --save

Updated Apt Sources
===================

You'll probably need to update the ``/etc/apt/sources.list`` file because the
default uses mirrors in Germany and doesn't include source URIs. Here's how it
looks:

::

    deb	 http://ftp2.de.debian.org/debian lenny main contrib non-free
    deb	 http://ftp2.de.debian.org/debian-security lenny/updates main contrib non-free

A good idea is to use the same settings as the base system. For me this means I
use these settings:

::

    deb http://ftp.uk.debian.org/debian/ lenny main
    deb-src http://ftp.uk.debian.org/debian/ lenny main
    
    deb http://security.debian.org/ lenny/updates main
    deb-src http://security.debian.org/ lenny/updates main
    
    deb http://volatile.debian.org/debian-volatile lenny/volatile main
    deb-src http://volatile.debian.org/debian-volatile lenny/volatile main

You'll need to update the packages and you might as well upgrade just to check
there aren't any new packages:

::

    $ sudo apt-get update
    $ sudo apt-get upgrade

Monitoring OpenVZ and Dealing with Problems
===========================================


If you are fairly new to OpenVZ and certain things that you expect to work
don't quite seem to, it is well worth running the ``sudo cat
/proc/user_beancounters`` command to get the OpenVZ status. If there are any
values which are not ``0`` in the last column this indicated a problem and you
should probably give the VE more resources.

::

    $ sudo cat  /proc/user_beancounters
    Version: 2.5
           uid  resource                     held              maxheld              barrier                limit              failcnt
            2:  kmemsize                  5215561              6133691             14372700             14790164                    0
                lockedpages                     0                    0                  256                  256                    0
                privvmpages                 43069                54099                65536                69632                    0
                shmpages                      640                  656                21504                21504                    0
                dummy                           0                    0                    0                    0                    0
                numproc                        44                   49                  240                  240                    0
                physpages                    7641                14347                    0  9223372036854775807                    0
                vmguarpages                     0                    0                33792  9223372036854775807                    0
                oomguarpages                 7641                14347                26112  9223372036854775807                    0
                numtcpsock                      9                   11                  360                  360                    0
                numflock                       12                   16                  188                  206                    0
                numpty                          1                    2                   16                   16                    0
                numsiginfo                      0                    4                  256                  256                    0
                tcpsndbuf                  185432               210568              1720320              2703360                    0
                tcprcvbuf                  147456                    0              1720320              2703360                    0
                othersockbuf                 6936               114064              1126080              2097152                    0
                dgramrcvbuf                     0                 4360               262144               262144                    0
                numothersock                   54                  120                  120                  120                   32
                dcachesize                 232872               334284              3409920              3624960                    0
                numfile                       881                 1012                 9312                 9312                    0
                dummy                           0                    0                    0                    0                    0
                dummy                           0                    0                    0                    0                    0
                dummy                           0                    0                    0                    0                    0
                numiptent                      10                   10                  128                  128                    0

Here ``failcnt`` indicated ``numothersock`` needs increasing.

Destroying a VE
===============

To destroy a VE you can use the following commands:

::

    $ sudo vzclt stop 221
    $ sudo vzclt destroy 221

