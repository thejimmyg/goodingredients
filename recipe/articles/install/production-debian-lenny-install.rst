Install Debian Lenny on a Production Server
+++++++++++++++++++++++++++++++++++++++++++

:Pre-Requisites: None
:Required Reading: None

.. include :: recipe/articles/install/starting-the-install.part

Partitioning Disks
==================

.. include :: recipe/articles/install/production-partitioning.part

.. include :: recipe/articles/install/continuing-the-base-install.part

Now add the second disk and reboot.

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

