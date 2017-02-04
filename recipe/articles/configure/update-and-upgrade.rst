Update and Upgrade
++++++++++++++++++

The first thing you need to do once you've logged in as ``root`` to a newly
installed Debian system with SSH is to update the list of packages available
and upgrade new packages to the latest versions. Of course, as you know, you
should never normally log in as root and when you configure SSH in the `Basic
SSH Security <basic-ssh-security.html>`_ article you'll disable root login over
SSH.

First check that the list in ``/etc/apt/sources.list`` is as you expect. If
you've installed Debian from an OpenVZ template for example it might need
updating to reflect a mirror nearby. Here's how it should look for a computer
in the UK:

::

    deb http://ftp.uk.debian.org/debian/ lenny main
    deb-src http://ftp.uk.debian.org/debian/ lenny main
    
    deb http://security.debian.org/ lenny/updates main
    deb-src http://security.debian.org/ lenny/updates main
    
    deb http://volatile.debian.org/debian-volatile lenny/volatile main
    deb-src http://volatile.debian.org/debian-volatile lenny/volatile main

Now update and upgrade your system as the ``root`` user like this:

::

    # apt-get update 
    # apt-get upgrade

The ``#`` character above represents the ``root`` user's prompt, you do not need to type it.

.. tip ::

   If you get this warning:

   ::

       W: GPG error: http://ftp.uk.debian.org lenny Release: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 9AA38DCD55BE302B
       W: There is no public key available for the following key IDs:
       9AA38DCD55BE302B
       W: You may want to run apt-get update to correct these problems


   You should run this:

   ::

       gpg --keyserver wwwkeys.eu.pgp.net --recv-keys 9AA38DCD55BE302B
       apt-key add ~/.gnupg/pubring.gpg
  
   Then run the commands again:

   ::
   
       apt-get update
       apt-get upgrade

You will now have a fully up-to-date system. If you have only just installed
Debian, you should already have the latest packages so don't be suprised if
there is nothing to update.

