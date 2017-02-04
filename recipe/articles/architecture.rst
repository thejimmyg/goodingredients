Server Architecture
+++++++++++++++++++

.. contents

The goal of the Good Ingredients server architecture is to provide a platform
which can reliably run all the services required by a modern web application in
such a way that all the services can be run on a dedicated server or moved
individually to their own servers to scale the service when needed.

As such the Good Ingredients server architecture provides an excellent basis on
which to build production web applications or enterprise applications.

Protocol-Level Proxy Architecture
=================================

The base architecture looks like this:

::

    Server A                            Server B
    -----------------------             -----------------------------

    Primary SMTP Mail Relay             Backup MX SMTP Mail Relay
    Master PostgreSQL A                 (Possible slave PostgreSQL B)
    Primary DNS Nameserver              Secondary DNS Nameserver
    HTTP Reverse Proxy A                HTTP Reverse Proxy B

* The DNS servers are set up to round-robin requests to both the HTTP Reverse
  Proxies so that both are utilised fairly evenly.
* The PostgreSQL server is used only to store data for the SMTP mail relays and
  other components, not as a general purpose server. You could add a slave on
  Server B and configure Postfix and Dovecot to use it if you wanted extra 
  resilience
* The HTTP reverse proxies handle HTTPS connections and proxy to backend HTTP
  servers

.. tip ::

    The architecture could also set up IMAP proxies to handle authentication
    and proxy to back-end IMAP servers.

These two servers together form a farily reliable way of forwarding email and
browser requests on to other services. They don't do anything useful in their
own right but provide a convenient protocol-level interface for dealing with
such requests.

Behind the PLPs you build the real servers which will run your applications.

Web Server Architecture
=======================

You can now set up any number of web servers and have their DNS entries
managed by the PLPs. More interestingly the HTTP proxy coukd also provide the
following services for you:

* HTTPS decoding
* GZIP encoding
* Caching of large uploads to help prevent simple DoS attacks
* Load balancing and failover
* Server side includes

Because all requests go through the PLPs it is easy to reconfigure them to send
requests elsewhere, allowing you to add extra back-end servers for capacity or
temporarily route requests to another server for maintainence.

Mail Servers
============

Many of the domains you'll set up will simply need mail forwarding rather than
hosting so that mail just goes to say your Gmail account. The PLPs will handle
email redirection for you and provide outgoing SMTP capabilities but if you
actually want to host mail to you can set up another server to receive the
email from the PLPs.

.. tip ::

    You can even use DRBD to duplicate data and setup multiple
    IMAP servers to serve the email. Once again by sending the requests through
    from the IMAP proxy you have a point at which to control how mail is handled.

SMTP failover is achived with the backup MX server.

Hardware Server
===============

Whilst ideally Server A and Server B should be on different machines in
different data centres for network and server fault-tolerence, there are still
some benefits to be gained by running them as two virtual servers on the same
physical machine rather than just having one server.  Namely that it allows you
to take down one of them for some purpose without breaking the rest of your
infrastrucutre - you just need to configure the nameservers to direct web and
IMAP services to the other machine and wait for the time to live to expire.

For a very simple setup, you might want to use virtual machines for each of the
servers so far. The choices are:

* Xen
* KVM
* OpenVZ

Redhat and Ubuntu have said that the future lies with KVM rather than Xen, so
although Xen is a great product there seems little point in using it. The
choice is then KVM or OpenVZ and boils down to this:

* KVM provides much stronger isolation between virtual machines (akin to Xen)
  but at the cost that unused resources on one virtual machine can't be used
  by others. It can run other operating systems such as Windows.
* OpenVZ uses a single kernel for all virtual machines and thus allows them 
  all to share memory at the expense of slightly weaker isolation. It cannot 
  run other operating system.

Since RAM is often the item at a premium in servers and since you are setting
up all the virtual machines for your own use and not giving access to
unpriveledge users on any of the virtual machines it seems to make sense to use
OpenVZ.

There is some more discussion in 
`KVM vs OpenVZ <kvm-vs-openvz-virtualization.html>`_.

Recommended Base Server Setup
=============================

When setting up each of the servers there are some very sensible defaults to
use:

* Debian Lenny (there's some more explaination of the choice of Debian in `Why Debian? <why-debian.html>`_
* Software RAID-1
* LVM2

Here are some benefits:

* By setting up RAID-1 and LVM2 on the base server, all the virtual 
  machines automatically benefit from it whilst being isolated from the 
  details. You can then easily take snapshots to back up individual virtual
  machines. By setting up an encrypted drive you can store sensitive data
  reliably.
* Since all data is proxied through the PLPs, you only need three external IP
  addresses. All other machines can be on a private network and have relevant
  ports (like SSH) forwarded on from the host
* You can setup an apt-cache on the host so that machines don't all have to 
  download the same updates.

Depending on your application you might also want to use:

* Encrypted Partitions

Start by following one of these tutorials:

* `Install Debian Lenny on a Production Server <install/production-debian-lenny-install.html>`_
* `Quick Installation on a Production Server <install/quick-installation.html>`_

You can also install everything in a virtualised environment for testing before a production deployment:

* `KVM Virtual Install of a Debian Lenny Production Setup <install/kvm-virtual-production-installation.html>`_

When you are done, you'll want to setup the Protocol Level Proxies:

* `Protocol Level Proxies <plps.html>`_

