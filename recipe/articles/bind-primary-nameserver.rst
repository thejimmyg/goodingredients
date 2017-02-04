Bind Primary Nameserver
+++++++++++++++++++++++

:Pre-Requisites: None
:Required Reading: None

.. contents ::

In this article (based partly on `this one
<http://www.dmo.ca/blog/20081009143754/>`_) we'll set up Bind 9 in a chroot to
serve as a the primary nameserver for a set of domains.

Whilst
there are extensions (such as `DLZ <http://bind-dlz.sourceforge.net/>`_) that
allow you to run BIND 9 from a MySQL database, for the number of domains I host
it is less hassle to use the standard config files. After all, we're not trying
to re-create `ISPConfig <http://www.ispconfig.org/>`_.

Masters and Slaves (Primary and Secondary Nameservers)
======================================================

There are two types of nameservers: primary nameservers (also called masters)
and secondary nameservers. Whilst a small cluster can probably get away with
one nameserver to serve DNS entries, most people choose to have at least two.
This is useful for handling large amounts of trafic and in case one nameserver
goes down. Some sites will have 5 or 6 nameservers or even more. To avoid
having to update each nameserver every time there is a change to one of the DNS
entries you make all but one of the nameservers *secondary nameservers* so that
they take their configuration from the primary. Slaves can tell when
information from a master has changed based on the serial numbers.

In our setup it makes sense to use PLP A as the primary and PLP B as the secondary. 

One server can be the Start of Authority (SOA) for one zone, while providing
secondary service for another zone, all the while providing caching services
for hosts on the local LAN. So as you can imagaine you can have some fairly complex 

Installation
============

Install BIND, and some tools to help with debugging:

::

    $ sudo apt-get install bind9 dnsutils

You could also install ``bind9-doc`` if you want the documentation.

If you run ``netstat`` you'll see BIND (known as ``named``) is running:

::

    $ sudo netstat -tap | grep named
    tcp        0      0 plpa.example.com:domain   *:*                     LISTEN      1028/named      
    tcp        0      0 localhost.locald:domain *:*                     LISTEN      1028/named      
    tcp        0      0 localhost.localdoma:953 *:*                     LISTEN      1028/named      
    tcp6       0      0 [::]:domain             [::]:*                  LISTEN      1028/named      
    tcp6       0      0 ip6-localhost:953       [::]:*                  LISTEN      1028/named   

Before we can put it in a chroot jail we need to shut it down:

::

    $ sudo /etc/init.d/bind9 stop
    Stopping domain name service...: bind9 waiting for pid 1028 to die.

Create the chroot directories and devices:

::

    sudo mkdir -p /var/chroot/bind9/{etc,dev,var/cache/bind,var/run/bind/run}
    sudo chown -R bind:bind /var/chroot/bind9/var/*
    sudo mknod /var/chroot/bind9/dev/null c 1 3
    sudo mknod /var/chroot/bind9/dev/random c 1 8
    sudo chmod 666 /var/chroot/bind9/dev/{null,random}

Move the BIND configuration directory and add a link so that any future Debian
updates will still work:

::

    sudo mv /etc/bind /var/chroot/bind9/etc
    sudo ln -s /var/chroot/bind9/etc/bind /etc/bind

Configuring Logging
--------------------

Even though BIND is now running in a chroot, we still want to see its log
events. Create ``/etc/rsyslog.d/bind-chroot.conf`` and add the line below (this
time you do need to add the ``$`` character too):

::

    $AddUnixListenSocket /var/chroot/bind9/dev/log

Restart syslogd:

::

    $ sudo /etc/init.d/rsyslog restart
    Restarting system log daemon: syslogd.

It should automatically create ``/dev/log`` inside the chroot:

::

    $ ls -la /var/chroot/bind9/dev/log
    srw-rw-rw- 1 root root 0 2009-05-25 14:38 /var/chroot/bind9/dev/log

Starting BIND 9
----------------

Edit ``/etc/default/bind9`` and replace this:

::

    OPTIONS="-u bind"

with this:

::

    OPTIONS="-u bind -t /var/chroot/bind9"

This tells BIND to use the chroot. Now start BIND:

::

    $ sudo /etc/init.d/bind9 start
    Starting domain name service...: bind.

and make sure it is running on the same ports as before:

::

    $ sudo netstat -tap | grep named
    tcp        0      0 plpa.example.com:domain   *:*                     LISTEN      1152/named      
    tcp        0      0 localhost.locald:domain *:*                     LISTEN      1152/named      
    tcp        0      0 localhost.localdoma:953 *:*                     LISTEN      1152/named      
    tcp6       0      0 [::]:domain             [::]:*                  LISTEN      1152/named      
    tcp6       0      0 ip6-localhost:953       [::]:*                  LISTEN      1152/named 

Finally check that the log messages get written to syslog:

::

    $ sudo tail -f -n 40 /var/log/syslog
    ...
    May 25 14:39:53 plpa named[1152]: starting BIND 9.5.1-P1 -u bind -t /var/chroot/bind9
    May 25 14:39:53 plpa named[1152]: found 2 CPUs, using 2 worker threads
    May 25 14:39:53 plpa named[1152]: using up to 4096 sockets
    May 25 14:39:53 plpa named[1152]: loading configuration from '/etc/bind/named.conf'
    May 25 14:39:53 plpa named[1152]: max open files (1024) is smaller than max sockets (4096)
    May 25 14:39:53 plpa named[1152]: using default UDP/IPv4 port range: [1024, 65535]
    May 25 14:39:53 plpa named[1152]: using default UDP/IPv6 port range: [1024, 65535]
    May 25 14:39:53 plpa named[1152]: listening on IPv6 interfaces, port 53
    May 25 14:39:53 plpa named[1152]: listening on IPv4 interface lo, 127.0.0.1#53
    May 25 14:39:53 plpa named[1152]: listening on IPv4 interface venet0:0, 192.168.100.2#53
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 254.169.IN-ADDR.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 2.0.192.IN-ADDR.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 255.255.255.255.IN-ADDR.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: D.F.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 8.E.F.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: 9.E.F.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: A.E.F.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: automatic empty zone: B.E.F.IP6.ARPA
    May 25 14:39:53 plpa named[1152]: command channel listening on 127.0.0.1#953
    May 25 14:39:53 plpa named[1152]: command channel listening on ::1#953
    May 25 14:39:53 plpa named[1152]: zone 0.in-addr.arpa/IN: loaded serial 1
    May 25 14:39:53 plpa named[1152]: zone 127.in-addr.arpa/IN: loaded serial 1
    May 25 14:39:53 plpa named[1152]: zone 255.in-addr.arpa/IN: loaded serial 1
    May 25 14:39:53 plpa named[1152]: zone localhost/IN: loaded serial 2
    May 25 14:39:53 plpa named[1152]: running

That's it. Now you need to set up zones.

Forward DNS, Reverse DNS and Zones
==================================

Before I explain how to configure BIND you need to know a bit of theory.

.. tip ::

    DNS for rocket scientists is a really excellent resource for learning about
    BIND and DNS. It contains all the details you need to if you get stuck I'd
    encourage you to look at it:

    * http://www.zytrax.com/books/dns/

As you are no doubt aware, DNS is used to find an IP address from a domain
name. This is called a *forward DNS lookup* and you might be used to
configuring ``A`` records for this purpose. You can also find out a domain name
from an IP address using a *reverse DNS lookup*. Reverse DNS entries rely on
``PTR`` records. For each domain name and IP address you are setting up you
need to configure both the forward and reverse DNS entries. 

A set of forward DNS entries for a particular domain is called a *zone*. A set
of reverse DNS entries forms a *reverse zone*. You need a zone and reverse zone
for every domain you wish to configure. The existence of the zones is specified
in your ``named.conf.local`` file but the entries themselves (called *resourse
records* are stored in two files, the forward entries go in db.domain where
``domain`` is replaced with the the domain name. For example ``db.example.com``
and the reverse DNS entries go into a file named based on the IP address
associated with the domain. For example ``db.192.168.100.in-addr.arpa``. For each A record
you configure in ``db.example.com`` you need to create a ``PTR`` record in
``db.192.168.100``. 

.. tip ::

    Technically you can name the zones anything you like, but the convention
    above helps to keep everything clear
  
Any time you make a change to a zone or reverse zone file you need to update
its serial number as this is what secondary nameservers use to determine
whether they need to update their own configurations.

Configuring BIND
================

BIND configuration is stored in the config files you've just moved to
``/var/chroot/bind9/etc/bind`` but you can access them at ``/etc/bind`` because
of the symlink you added.

Here are the files and a decription of what they are for:

``named.conf``

    This is the main configuration file for BIND. It imports the options in
    ``named.conf.options`` and ``named.conf.local``. It also sets up zones for
    ``localhost``, ``127.in-addr.arpa``, ``0.in-addr.arpa``, ``255.in-addr.arpa``
    and for the root nameservers.

    The idea is that you'll edit ``named.conf.options`` and
    ``named.conf.local`` to customise BIND and use this file alone

``named.conf.options``

    This file contains settings controlling how the BIND server itself works

``named.conf.local``

    This file contains definitions for all the zones you are setting up.

``db.0``, ``db.255``, ``db.local``, ``db.127``, ``db.root``

    These files all contain the DNS entries used by the zones defined in
    ``named.conf``. You should leave these files alone.  The root servers change
    over time, so ``db.root`` must be maintained now and then but this is usually
    done automatically as updates to bind.

``zones.rfc1918``

    This contains zone entries for the private networks described in http://www.faqs.org/rfcs/rfc1918.html.

    * 10.0.0.0        -   10.255.255.255  (10/8 prefix)
    * 172.16.0.0      -   172.31.255.255  (172.16/12 prefix)
    * 192.168.0.0     -   192.168.255.255 (192.168/16 prefix)

    These zones are not inluded by default because the line is commented out in ``named.conf.local``.

``db.empty``
    This contains empty reverse DNS entries used by the zones defined in ``zones.rfc1918``

``rndc.key``
    A key which allows the rndc commands to work

See also:

* https://help.ubuntu.com/8.10/serverguide/C/dns.html

Updating ``named.conf.options``
-------------------------------

With the comments removed the ``/etc/bind/named.conf.options`` file starts out like this:

::

    options {
            directory "/var/cache/bind";
            auth-nxdomain no;    # conform to RFC1035
            listen-on-v6 { any; };
    };

It's a good idea to add a couple more options between the curly braces:

::

           version none;
           allow-transfer { none; };

The ``version none;`` option means BIND won't tell you its version when you
ask. This is useful because it prevents a potential attacker easily working out
which attack to use.

The ``allow-transfer { none; };`` option prevents other servers from
downloading zone information. You'll change this if you set up a slave
nameserver.

.. tip ::

   BIND is rather fussy when it comes to syntax and won't start if you make a
   mistake. Every statement must end with a semi-colon which is why there are
   ``;`` characters before *and* after the ``}`` characters in the options shown
   above.

Updating ``named.conf.local``
-----------------------------

This is the file where you add your zone entries. Let's set up the domains
``plpa.example.com``, ``plpb.example.com`` and ``maila.example.com`` so that we can
test the mail servers properly.

These domains are all part of the ``example.com`` zone.

Let's add an entry for example.com ``/etc/bind/named.conf.local``:

::

    zone "example.com" {
        type master;
        file "/etc/bind/db.example.com";
    };

Let's also add an entry for the reverse zone:

::

    zone "122.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.122.168.192.in-addr.arpa";
    };

.. tip ::

   Notice that we're creating the reverse zone for ``122.168.192.in-addr.arpa``
   rather than ``2.122.168.192.in-addr.arpa``. This allows us to put all the
   entries for the ``192.168.100.*``.

   Note that if you are using a server where the subnet is owned by the hosting
   provider it is likely they will set up these reverse DNS entries instead since
   they will own the subnet. They will probably provide a web based interface to
   allow you to update the reverse DNS entries or their tech support people should
   be able to set them up.

Now we need to create the zone files themselves.

A Sample Forward Zone
---------------------

Here's an example zone file for the example.com domain. I haven't used any of the
shortcuts you might have seen so what I've written here is pretty much what
you'll get in the Save it as ``/etc/bind/db.example.com``:

::

    $TTL 84924 ; zone default = 1 day or 84924 seconds
    $ORIGIN example.com.
    example.com.    IN      SOA   ns0.example.com. example.com. (
        2009052300 ; serial number
        3h         ; refresh =  3 hours 
        15M        ; update retry = 15 minutes
        1W12h      ; expiry = 1 week + 12 hours
        2h20M      ; minimum = 2 hours + 20 minutes
    )
    example.com.       84924  IN  NS  ns0.example.com. 
    example.com.       84924  IN  NS  ns1.example.com.
    example.com.       84924  IN  MX  10  plpa.example.com.
    example.com.       84924  IN  MX  20  plpb.example.com.
    example.com.       84924  IN  A   192.168.100.141
    plpa.example.com.  84924  IN  A   192.168.100.2
    plpb.example.com.  84924  IN  A   192.168.100.3
    maila.example.com. 84924  IN  A   192.168.100.4
    ns0.example.com.   84924  IN  A   192.168.100.2
    ns1.example.com.   84924  IN  A   192.168.100.3
    www.example.com.   84924  IN  A   192.168.100.2 ; Let's use DNS round robin for the www sub-domain
    www.example.com.   84924  IN  A   192.168.100.3

For an explanation of these records you'll need to read these two guides:

* `http://debianclusters.cs.uni.edu/index.php/Forward_DNS_-_Name_to_IP_Address_(db.yourdomain)`
* http://www.zytrax.com/books/dns/ch8/

A Sample Reverse Zone
---------------------

Now a reverse DNS zone. Save it as ``/etc/bind/db.2.122.168.192.in-addr.arpa``:

::

    $TTL 84924 ; zone default = 1 day or 84924 seconds
    $ORIGIN example.com.
    2.122.168.192.in-addr.arpa.   IN      SOA   ns0.example.com. example.com. (
        2009052300 ; serial number
        3h         ; refresh =  3 hours 
        15M        ; update retry = 15 minutes
        1W12h      ; expiry = 3 weeks + 12 hours
        2h20M      ; minimum = 2 hours + 20 minutes
    )
                                  IN    NS   ns0.example.com. 
                                  IN    NS   ns1.example.com.
    141.122.168.192.in-addr.arpa. IN   PTR   example.com.
    2.122.168.192.in-addr.arpa.   IN   PTR   plpb.example.com.
    3.122.168.192.in-addr.arpa.   IN   PTR   plpa.example.com.
    4.122.168.192.in-addr.arpa.   IN   PTR   maila.example.com.

Restarting Bind
===============

Once all the changes are made restart bind. 

::

    $ sudo /etc/init.d/bind9 restart

Then check there haven't been any problems by looking syslog:

::

    $ sudo tail -f /var/log/syslog

The output should include these lines:

::

    May 25 19:49:57 plpa named[761]: zone 122.168.192.in-addr.arpa/IN: loaded serial 2009052300
    May 25 19:49:57 plpa named[761]: zone example.com/IN: loaded serial 2009052300
    May 25 19:49:57 plpa named[761]: zone localhost/IN: loaded serial 2
    May 25 19:49:57 plpa named[761]: zone example.com/IN: sending notifies (serial 2009052300)
    May 25 19:49:57 plpa named[761]: running

Testing DNS
===========

If everything is working you should be able to dig the different domains set up:

::

    $ dig @localhost www.example.com 
    ; <<>> DiG 9.5.1-P1 <<>> @localhost www.example.com
    ; (1 server found)
    ;; global options:  printcmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13439
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 2
    
    ;; QUESTION SECTION:
    ;www.example.com.			IN	A
    
    ;; ANSWER SECTION:
    www.example.com.		84924	IN	A	192.168.100.2
    www.example.com.		84924	IN	A	192.168.100.3
    
    ;; AUTHORITY SECTION:
    example.com.		84924	IN	NS	ns0.example.com.
    example.com.		84924	IN	NS	ns1.example.com.
    
    ;; ADDITIONAL SECTION:
    ns0.example.com.		84924	IN	A	192.168.100.2
    ns1.example.com.		84924	IN	A	192.168.100.3
    
    ;; Query time: 4 msec
    ;; SERVER: 127.0.0.1#53(127.0.0.1)
    ;; WHEN: Mon May 25 19:51:05 2009
    ;; MSG SIZE  rcvd: 131

You can check reverse DNS like this:

::

    $ nslookup 192.168.100.4
    Server:		192.168.100.1
    Address:	        192.168.100.1#53

    4.122.168.192.in-addr.arpa	    name = maila.example.com.

Setting up Domains with the Nameserver
======================================

Certain domain *registrars* will require that nameservers are registered with
the *registry* before they can be used with their control panels. This means
you will need to create what is known as a "Glue Record" (`glue records are
described here
<https://www.dyndns.com/support/kb/what_is_domain_registration.html#glue>`_).
Technically these are only really needed if the nameserver domain for a domain
is under the domain itself because without a glue record specifying the IP
address explicitly there would be no way to resolve the nameserver.

Making servers use this DNS server
==================================

Once you have made sure BIND is working correctly you change all of the
machines on the network to use the new nameserver instead of the current one.
This is very useful for testing as it means you can use internal IP addresses
(as we've done in this article) and the machines will use these rather than the
values live on the internet. You'd probably want to skip this for production
use though. 

To make the change edit the file ``/etc/resolv.conf`` and replace the content with:

::

    nameserver 192.168.100.2

Once this change is made you won't need to add the ``@localhost`` to the
``dig`` commands because they will use localhost anyway.

Testing Round-Robin
===================

You can now test the DNS round robin for the www subdomain like this:

::

    $ ping www.example.com
    PING www.example.com (192.168.100.2) 56(84) bytes of data.
    64 bytes from plpa.example.com (192.168.100.2): icmp_seq=1 ttl=64 time=0.075 ms
    64 bytes from plpa.example.com (192.168.100.2): icmp_seq=2 ttl=64 time=0.061 ms
    ^C
    --- www.example.com ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1000ms
    rtt min/avg/max/mdev = 0.061/0.068/0.075/0.007 ms
    $ ping www.example.com
    PING www.example.com (192.168.100.3) 56(84) bytes of data.
    64 bytes from plpb.example.com (192.168.100.3): icmp_seq=1 ttl=64 time=0.110 ms
    64 bytes from plpb.example.com (192.168.100.3): icmp_seq=2 ttl=64 time=0.085 ms
    ^C
    --- www.example.com ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 999ms
    rtt min/avg/max/mdev = 0.085/0.097/0.110/0.015 ms

Notice that the first ping went to 192.168.100.2 but the second went to
192.168.100.3.

One problem though is what happens if a you need to resolve a domain which
isn't one of the domains in the BIND config files? Let's try it:

::

    $ ping google.com
    ping: unknown host google.com

The lookup fails. To get this to work you need to set up BIND as a *caching
nameserver*.

