Times and Dates
+++++++++++++++

There are two types of date and time on Debian: local time which is the time in
the current timezone and coordinated universal time (UTC) which is pretty much
equivalent to Grenwich Mean Time at any time (give or take a few seconds) but
which can be used everywhere.

You can get the current date, time and timezone of the system's local time zone with
the ``date`` command:

::

    $ date
    Sat May 16 14:26:26 BST 2009

Here you can see that the server is using the British Summer Time time zone.

To see the UTC time you use this:

::

    $ date
    Sat May 16 13:26:26 UTC 2009

You can also set the local date and time with the ``date`` command:

::

    $ date 051614472009.23

This is in the format ``MMDDhhmm[[CC]YY][.ss]`` so the above command sets the
date to May 16th 14:47:23 2009

Run ``man date`` to see the details.

Changing Time Zone
==================

If you are running more than one server in more than one time zone it is a good idea to use UTC so that all servers share the same local time.

You can change the timezone with this command:

::

    $ sudo dpkg-reconfigure tzdata

You'll need to chose ``Etc`` and then ``UTC`` from the two menus
which appear. Then you'll then see the following output:

::

    Current default timezone: 'Etc/UTC'
    Local time is now:      Sat May 16 13:42:36 UTC 2009.
    Universal Time is now:  Sat May 16 13:42:36 UTC 2009.

Making Sure the Time is Correct
===============================

Just because the time and date is correct now doesn't mean it will be in the
future. All sorts of factors can cause the time on your system to gradually
become incorrect. In order to ensure your server uses the same time as others
you can use the Network Time Protocol (NTP) to syncronise the time with other
computers on the internet.

.. note ::

   Setting up NTP is actually very important, it is easy for slightly out of
   sync clocks to cause unexpected problems.

Install NTP like this:

::

    $ sudo apt-get install ntp ntpdate

This will also install a script ``/etc/init.d/ntp`` which will start the ``ntpd`` daemon to automatically set the time.

You can check the peers with:

::

    $ ntpq -p
         remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
     ntp.ubuntu.com  193.79.237.14    2 u   35   64    1   30.217  -484.91   0.001

Here you can see the ``ntp.ubuntu.com`` server is being used to set the time.

If you don't install the NTP daemon you can set the date using the ``ntpdate`` command:

::

    $ sudo ntpdate ntp.ubuntu.com

Virtualized Setup
=================

In a virtualized setup the time of the guest is usually set by the host. To allow a guest and host to have different times you need to make some changes on the host.

Xen
---

For Xen:

First run this in the DomU::

    echo 1 > /proc/sys/xen/independent_wallclock

Then edit ``/etc/sysctl.conf`` and add this to the end to make it permanant::

    cat <<EOF>> /etc/sysctl.conf
    xen.independent_wallclock = 1
    EOF

Now the DomU clock can be independant of the Dom0 clock. 

KVM
---

The host and guest are independent unless you use the kvmclock clocksource.

OpenVZ
------

In OpenVZ, VEs share the host's system time. To allow the clocks to differ the VE needs to be given the CAP_SYS_TIME capability:

::

    vzctl set 101 --capability sys_time:on --save

See also:

* http://en.wikipedia.org/wiki/Coordinated_Universal_Time
* http://en.wikipedia.org/wiki/Network_Time_Protocol
