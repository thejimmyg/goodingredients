Basic SSH Security
++++++++++++++++++

Since the server is going to be on the internet anyone else on the internet can
access it so it is important it is as secure as possible. To try to prevent a
malicious user gaining root access to the system by guessing the root password
we will do a couple of things:

* Disable the root login (instead you can login as ``owner`` or whichever user
  you gave sudo access to in `Add a User and Configure sudo
  <add-a-user-and-configure-sudo.html>`_ and then use the ``sudo`` command)

* Put the SSH server on a non-standard port so it is slightly harder to find

* Define which users can log in

Make a backup of ``/etc/ssh/sshd_config`` and then check or change the following:

The main things to change (or check) are below. Note that if you get them wrong
you might not be able to login to the machine remotely so be careful. 

::

    Port 28500
    PermitRootLogin no
    UsePAM no
    X11Forwarding no
    UseDNS no
    AllowUsers owner

Above we've used port 28500 but you should change this to another port with a
large number. Write down the port you've chosen because it can be easy to
forget. 

The ``AllowUsers`` line should list whichever user you set up when configuring
sudo. In the example the user was called ``owner`` which is what is used here.
If you later create another account and want to let someone else sign in as
well as ``owner`` you'll need to update the ``AllowUsers`` option. Multiple
users can be specified, separated by spaces. Here we allow both the users
``owner`` and ``dev`` to sign in over SSH:

::

    AllowUsers owner dev

Once you happy with the settings restart:

::

    $ sudo /etc/init.d/ssh restart
    Restarting OpenBSD Secure Shell server: sshd.

Don't exit that shell though before you've loaded up another terminal and
checked you can connect again. This time you'll need to use this:

::

    ssh owner@server1.example.com -p 28500


.. tip :: 

    If you do make a mistake and can't reconnect and are fortunate enough to be
    using a host with a rescue system, boot into the rescue system and mount the
    drive containing the ``/etc/`` directory, in this example it is on
    ``/dev/sda2``:
    
    ::

        # mount /dev/sda2 /mnt -t ext3
    
    You will then be able to edit the file as ``/mnt/etc/ssh/sshd_config`` and then
    reboot back into your normal setup.
    
    If you are using LVM it is a little more complex, you must do this to mount the drive:

    ::

        # lvdisplay | grep "LV Name"
          LV Name                /dev/vg_root/root
          LV Name                /dev/vg_root/swap
          LV Name                /dev/vg_root/tmp

    Use the volume group name, in this case ``vg_root`` in the next command:

    ::

        # vgchange -a y vg_root
          3 logical volume(s) in volume group "vg_root" now active

    Now mount the logical volume:

    ::
    
        # mount /dev/vg_root/root /mnt

See also:

* http://www.ssh.com/support/documentation/online/ssh/adminguide/32/Restricting_User_Logins.html


Port Forwarding
================

If you are setting up SSH on a private VE which isn't on a publically
accessible IP address you will need to set up port forwarding on the HE. In
this case there isn't much point in changing the default port because it can't
be accessed anyway. Instead you should change the firewall on the HE to forward
a spare port to the VE on port 22. For example, to forward port 30004 on the HE
to port 22 on a VE with IP address 192.168.100.4 you would run this on the HE:

::

    sudo iptables -t nat -A PREROUTING -p tcp -d 188.40.40.131 --dport 30004 -i eth0 -j DNAT --to-destination 192.168.100.4:22

If you have `set up your firewall <firewall.html>`_ you can make this change
permanant by editing the ``/etc/iptables.rules`` file and change these lines in the ``*nat`` section:

::

    :OUTPUT ACCEPT [26:1976]
    -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j SNAT --to-source 188.40.40.131
    COMMIT

to this:

::

    :OUTPUT ACCEPT [26:1976]
    -A PREROUTING -d 188.40.40.131/32 -i eth0 -p tcp -m tcp --dport 30004 -j DNAT --to-destination 192.168.100.4:22
    -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j SNAT --to-source 188.40.40.131
    COMMIT


