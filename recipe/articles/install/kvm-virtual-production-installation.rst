KVM Virtual Install of a Debian Lenny Production Setup
++++++++++++++++++++++++++++++++++++++++++++++++++++++

:Pre-Requisites: None
:Required Reading: None

Even though the Good Ingredients servers will eventually run OpenVZ (as explained in the `KVM vs OpenVZ <kvm-vs-openvz-virtualization.html>`_ article) it can be useful to set up KVM on a desktop machine so you can test different OpenVZ configurations (or indeed other server setups) on your local machine before deploying them to a production environment. 

In this article you'll learn how to set up KVM and link a virtual machine to the Debian 5.0 (Lenny) net install CD so that the virtual machine will boot from the CD.

.. include :: recipe/articles/install/using-virtual-machine-manager.part

.. include :: recipe/articles/install/starting-the-install.part

Partitioning Disks
==================

.. include :: recipe/articles/install/production-partitioning.part

.. include :: recipe/articles/install/continuing-the-base-install.part

.. include :: recipe/articles/install/add-a-second-virtual-disk.part

.. include :: recipe/articles/install/finish-base-encrypt-install.part

.. include :: recipe/articles/install/set-up-openssh.part

.. include :: recipe/articles/install/set-up-raid-1.part

It worked! More resources:

* http://linux-raid.osdl.org/index.php/Growing
* http://www.howtoforge.com/perfect-server-debian-lenny-ispconfig3
* http://www.howtoforge.com/set-up-a-fully-encrypted-raid1-lvm-system-p6
* http://en.wikipedia.org/wiki/Logical_disk
* http://en.wikipedia.org/wiki/Standard_RAID_levels
* http://www.linuxdevcenter.com/pub/a/linux/2006/04/27/managing-disk-space-with-lvm.html
* http://www.ducea.com/2009/03/08/mdadm-cheat-sheet/
* http://en.gentoo-wiki.com/wiki/DM-Crypt

