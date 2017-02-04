Bind Secondary Nameserver
+++++++++++++++++++++++++

:Pre-Requisites: None
:Required Reading: None

.. contents ::

Make sure you've followed this tutorial to the end for the primary nameserver:

* `Bind Primary Nameserver <bind-primary-nameserver.html>`_

A secondary nameserver can get the zone information from a primary which means
you only have to update the primary when you want to make changes. Having more
than one nameserver provides some resilience and can help deal with high loads.

Installing BIND
===============

The initial install of BIND into a chroot is the same for a secondary
nameserver as for a primary nameserver. Rather than going over it again in
detail we'll just summarise the instructions here:

Install BIND and some tools to help with debugging:

::

    $ sudo apt-get install bind9 dnsutils

Stop BIND:

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

Create ``/etc/rsyslog.d/bind-chroot.conf`` and add the line:

::

    $AddUnixListenSocket /var/chroot/bind9/dev/log

Restart syslogd:

::

    $ sudo /etc/init.d/rsyslog restart
    Restarting system log daemon: syslogd.

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

If you have any problems, go back and follow the more detailed steps in the
primary nameserver article.

Changes to the Primary Nameserver
=================================

You need to specify the IP address of the slave servers in the primary
nameserver's ``/etc/bind/named.conf.options`` file before the master will allow
another nameserver to access its zone information. Update the
``allow-transfer`` option to look like this:

::

    allow-transfer { 192.168.100.3; };

Then restart bind:

::

    $ sudo /etc/init.d/bind9 restart

and check ``/var/log/syslog`` to ensure everything worked correctly:

::

    $ sudo tail -f /var/log/syslog
    May 25 22:33:45 plpa named[959]: command channel listening on ::1#953
    May 25 22:33:45 plpa named[959]: zone 0.in-addr.arpa/IN: loaded serial 1
    May 25 22:33:45 plpa named[959]: zone 127.in-addr.arpa/IN: loaded serial 1
    May 25 22:33:45 plpa named[959]: zone 122.168.192.in-addr.arpa/IN: loaded serial 2009052300
    May 25 22:33:45 plpa named[959]: zone 255.in-addr.arpa/IN: loaded serial 1
    May 25 22:33:45 plpa named[959]: zone example.com/IN: loaded serial 2009052300
    May 25 22:33:45 plpa named[959]: zone localhost/IN: loaded serial 2
    May 25 22:33:45 plpa named[959]: zone example.com/IN: sending notifies (serial 2009052300)
    May 25 22:33:45 plpa named[959]: zone 122.168.192.in-addr.arpa/IN: sending notifies (serial 2009052300)
    May 25 22:33:45 plpa named[959]: running

Configuring the Secondary Nameserver
====================================

Unlike on the primary, you don't need to create any ``db.*`` files, instead you
just specify the zones in ``/etc/bind/named.conf.local`` but within those
definitions, specify the ``type`` as ``slave`` and add the IP address of the
master from which those files can be obtained. Edit
``/etc/bind/named.conf.local`` and add this to the end:

::

    zone "example.com" {
        type slave;
        file "/etc/bind/db.example.com";
     	masters {
            192.168.100.2;
     	};
    };

Let's also add an entry for the reverse zone:

::

    zone "122.168.192.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.122.168.192.in-addr.arpa";
     	masters {
            192.168.100.2;
     	};
    };

With these changes in place, reload the configuration:

:: 

    $ sudo /etc/init.d/bind9 reload

If you look at ``/var/log/syslog`` you'll see that the zone transfer failed because of a permission problem

::

    $ sudo tail -f /var/log/syslog 
    ...
    May 25 22:45:15 plpb named[1447]: zone example.com/IN: Transfer started.
    May 25 22:45:15 plpb named[1447]: transfer of 'example.com/IN' from 192.168.100.2#53: connected using 192.168.100.3#41069
    May 25 22:45:15 plpb named[1447]: dumping master file: /etc/bind/tmp-4h0Q5PEDYM: open: permission denied
    May 25 22:45:15 plpb named[1447]: transfer of 'example.com/IN' from 192.168.100.2#53: failed while receiving responses: permission denied
    May 25 22:45:15 plpb named[1447]: transfer of 'example.com/IN' from 192.168.100.2#53: Transfer completed: 0 messages, 14 records, 0 bytes, 0.030 secs (0 bytes/sec)
    ...    

To fix this you need to set the group of the symlink ``/etc/bind`` to be ``bind``, give the group right access and then reload:

::

    $ sudo chgrp --no-dereference bind /etc/bind
    $ sudo chmod g+w /etc/bind
    $ sudo /etc/init.d/bind9 reload

This time the transfers are successful:

::

    james@plpb:~$ sudo tail -f /var/log/syslog 
    ...
    May 25 22:56:21 plpb named[1508]: zone 122.168.192.in-addr.arpa/IN: Transfer started.
    May 25 22:56:21 plpb named[1508]: transfer of '122.168.192.in-addr.arpa/IN' from 192.168.100.2#53: connected using 192.168.100.3#41738
    May 25 22:56:21 plpb named[1508]: zone 122.168.192.in-addr.arpa/IN: transferred serial 2009052300
    May 25 22:56:21 plpb named[1508]: transfer of '122.168.192.in-addr.arpa/IN' from 192.168.100.2#53: Transfer completed: 1 messages, 8 records, 241 bytes, 0.002 secs (120500 bytes/sec)
    ...

From now on, each time you make a change to the zone files on the primary and
increment the serial number, bind sends a NOTIFY to all secondaries. The
secondaries then check to see if the serial number matches the files they
already have and if not they perform a zone transfer to get the updated zone
information.

The only time you should need to modify the settings on the secondary is when
you've added a new zone on the primary which the secondary isn't aware of.

Testing
=======

You can test this nameserver as you did with the primary:

::

    $ dig @localhost www.example.com
    
    ; <<>> DiG 9.5.1-P1 <<>> @localhost www.example.com
    ; (1 server found)
    ;; global options:  printcmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38231
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 2
    
    ;; QUESTION SECTION:
    ;www.example.com.			IN	A
    
    ;; ANSWER SECTION:
    www.example.com.		84924	IN	A	192.168.100.2
    www.example.com.		84924	IN	A	192.168.100.3
    
    ;; AUTHORITY SECTION:
    example.com.		84924	IN	NS	ns1.example.com.
    example.com.		84924	IN	NS	ns0.example.com.
    
    ;; ADDITIONAL SECTION:
    ns0.example.com.		84924	IN	A	192.168.100.2
    ns1.example.com.		84924	IN	A	192.168.100.3
    
    ;; Query time: 4 msec
    ;; SERVER: 127.0.0.1#53(127.0.0.1)
    ;; WHEN: Mon May 25 22:58:41 2009
    ;; MSG SIZE  rcvd: 131

Now edit ``/etc/bind/db.example.com`` on the master and add a new domain by adding this line at the end:

::

    new.example.com.		84924	IN	A	192.168.100.5

Reload the master:

::

    $ sudo /etc/init.d/bind9 reload

On the secondary you'll see these messages in ``/var/log/syslog``:

::

    May 25 23:03:55 plpb named[1508]: client 192.168.100.2#27684: received notify for zone 'example.com'
    May 25 23:03:55 plpb named[1508]: zone example.com/IN: notify from 192.168.100.2#27684: zone is up to date
    May 25 23:03:56 plpb named[1508]: client 192.168.100.2#54635: received notify for zone '122.168.192.in-addr.arpa'
    May 25 23:03:56 plpb named[1508]: zone 122.168.192.in-addr.arpa/IN: notify from 192.168.100.2#54635: zone is up to date

The secondary has been notified of the change but thinks the changed zone is up to date because we didn't update the serial number. You can check this for yourself by running this on the secondary:

::

    $ dig @localhost new.example.com

You won't get an ``ANSWER SECTION`` because the DNS entry doesn't exist on the secondary yet. 

Update the serial number on the master and reload:

::

    $ sudo /etc/init.d/bind9 reload

On the secondary you'll see these messages in ``/var/log/syslog``:

::

    May 25 23:07:08 plpb named[1508]: client 192.168.100.2#21504: received notify for zone 'example.com'
    May 25 23:07:08 plpb named[1508]: zone example.com/IN: Transfer started.
    May 25 23:07:08 plpb named[1508]: transfer of 'example.com/IN' from 192.168.100.2#53: connected using 192.168.100.3#33030
    May 25 23:07:08 plpb named[1508]: zone example.com/IN: transferred serial 2009052301
    May 25 23:07:08 plpb named[1508]: transfer of 'example.com/IN' from 192.168.100.2#53: Transfer completed: 1 messages, 15 records, 335 bytes, 0.003 secs (111666 bytes/sec)
    May 25 23:07:08 plpb named[1508]: zone example.com/IN: sending notifies (serial 2009052301)
    May 25 23:07:09 plpb named[1508]: client 192.168.100.2#12845: received notify for zone '122.168.192.in-addr.arpa'
    May 25 23:07:09 plpb named[1508]: zone 122.168.192.in-addr.arpa/IN: notify from 192.168.100.2#12845: zone is up to date

Notice this time the DNS entries were updated and ``dig`` returns the new record:

::

    $ dig @localhost new.example.com
    ...
    ;; ANSWER SECTION:
    new.example.com.		84924	IN	A	192.168.100.5
    ...

That's it, you now have primary and secondary nameservers.
