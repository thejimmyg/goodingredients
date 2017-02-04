Add a User and Configure sudo
+++++++++++++++++++++++++++++

Now that your system is up-to-date you should configure ``sudo``. ``sudo`` is a program which lets an ordinary user of the system gain root priveledges under certain conditions. 

As you know you never log in as the root user. It is considered safer to always perform administration as a non-priviledged user, only using root priviledges when you really need them. When you configure SSH in the `Basic SSH Security <basic-ssh-security.html>`_ article you'll disable root login over SSH.

During the installation of Debian you set up a non-priveldeged user. The example showed a user with the name ``Debian User`` and the username ``debian``. You can use the user you set up during installation if you prefer or create another user. Here I'm creating another user called ``owner``:

As ``root``, run this:

::

    # adduser owner
    Adding user `owner' ...
    Adding new group `owner' (1000) ...
    Adding new user `owner' (1000) with group `owner' ...
    Creating home directory `/home/owner' ...
    Copying files from `/etc/skel' ...
    Enter new UNIX password: 
    Retype new UNIX password: 
    passwd: password updated successfully
    Changing the user information for owner
    Enter the new value, or press ENTER for the default
    	Full Name []: 
    	Room Number []: 
    	Work Phone []: 
    	Home Phone []: 
    	Other []: 
    Is the information correct? [Y/n] y

Once the user is set up install ``sudo``.

::

    # apt-get install sudo

Now issue the ``visudo`` command to configure ``sudo``:

::

    # visudo

At the end of the file add this line:

::

    owner   ALL=(ALL) ALL

This gives the ``owner`` user full root privelidges when using ``sudo``.

You can now sign out of SSH by typing ``exit`` and sign in again as whichever user you've set up:

::

    ssh owner@server1.example.com

You can now run commands with ``root`` priviledges by prefixing the command with ``sudo``. The first time you run a command you'll be prompted for the ``owner`` user's password (or if you haven't entered a command with ``sudo`` for a few minutes). Here's an example where we update the packages again, this time as ``owner``:

::

    $ sudo apt-get update

You don't type the ``$``, this is just there to make it clear you are running as a non-root user.

From now on you should need log in as the ``root`` user.

