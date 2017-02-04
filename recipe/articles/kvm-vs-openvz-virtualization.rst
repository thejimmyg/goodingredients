KVM vs OpenVZ Virtualization
++++++++++++++++++++++++++++

KVM has better isolation, OpenVZ has better performance.

This article from the OpenVZ website explains the differences:
http://wiki.openvz.org/Introduction_to_virtualization

You won't be able to run anything other than Linux when using OpenVZ, but that
is exactly what we want to do anyway.

One solution called `Proxmox VE <http://pve.proxmox.com/wiki/Main_Page>`_
allows OpenVZ and KVM to work together on the same machine giving you the
advantages of each. It is Open Source and based on Debian Lenny so you may wish
to investigate it.
