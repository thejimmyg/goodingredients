Spam and Virus Protection
+++++++++++++++++++++++++

Some estimate calculate that spam averages 94% of all e-mail sent and without
good spam protection it can make up a significant proportion of a user's
emails. 

Luckily there is good open-source spam protection software available and this
tutorial will show you how to use it.

Which Servers to Place the Protection On
========================================

One of the problems with spam and virus protection is that it can be fairly
resource-intesnive as each mail has to be compared and rated against various
databases and factors. Whilst you could put the spam protection on the PLP A
and PLP B servers to filter out spam as soon as it reaches your servers, I
prefer to filter it on the main mail nodes such as mail A.

Install amavisd-new, SpamAssassin, And ClamAV
=============================================

AMaViS is a mail virus scanner which you can tell Postfix to pipe emails
through. Amavis can then check the emails with spamassassin to check for spam
and clam AV to check for viruses.

Install the three programs like this:

::

    $ sudo apt-get install amavisd-new spamassassin clamav clamav-daemon pax

Other tutorials suggest you need these too but I've never installed them:

::

    $ sudo apt-get install zoo unzip bzip2 libnet-ph-perl libnet-snpp-perl libnet-telnet-perl nomarch lzop pax

Afterwards we must configure ``amavisd-new``. The configuration is split up in various files which reside in the ``/etc/amavis/conf.d`` directory. Take a look at each of them to become familiar with the configuration. Most settings are fine, however we must modify three files.

First we must enable ClamAV and SpamAssassin in ``/etc/amavis/conf.d/15-content_filter_mode`` by uncommenting the ``@bypass_virus_checks_maps`` and the ``@bypass_spam_checks_maps`` lines:

The file should look like this:

::

    use strict;
    
    # You can modify this file to re-enable SPAM checking through spamassassin
    # and to re-enable antivirus checking.
    
    #
    # Default antivirus checking mode
    # Uncomment the two lines below to enable it back
    #
    
    @bypass_virus_checks_maps = (
       \%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);
    
    
    #
    # Default SPAM checking mode
    # Uncomment the two lines below to enable it back
    #
    
    @bypass_spam_checks_maps = (
       \%bypass_spam_checks, \@bypass_spam_checks_acl, \$bypass_spam_checks_re);
    
    1;  # ensure a defined return

Now take a look at the spam and virus settings in ``/etc/amavis/conf.d/20-debian_defaults``. There's no need to change anything if the default settings are ok for you. The file contains many explanations so there's no need to explain the settings here:

::

    ...
    $QUARANTINEDIR = "$MYHOME/virusmails";
    $quarantine_subdir_levels = 1; # enable quarantine dir hashing
    
    $log_recip_templ = undef;    # disable by-recipient level-0 log entries
    $DO_SYSLOG = 1;              # log via syslogd (preferred)
    $syslog_ident = 'amavis';    # syslog ident tag, prepended to all messages
    $syslog_facility = 'mail';
    $syslog_priority = 'debug';  # switch to info to drop debug output, etc
    
    $enable_db = 1;              # enable use of BerkeleyDB/libdb (SNMP and nanny)
    $enable_global_cache = 1;    # enable use of libdb-based cache if $enable_db=1
    
    $inet_socket_port = 10024;   # default listening socket
    
    $sa_spam_subject_tag = '***SPAM*** ';
    $sa_tag_level_deflt  = 2.0;  # add spam info headers if at, or above that level
    $sa_tag2_level_deflt = 6.31; # add 'spam detected' headers at that level
    $sa_kill_level_deflt = 6.31; # triggers spam evasive actions
    $sa_dsn_cutoff_level = 10;   # spam level beyond which a DSN is not sent
    ...
    $final_virus_destiny      = D_DISCARD;  # (data not lost, see virus quarantine)
    $final_banned_destiny     = D_BOUNCE;   # D_REJECT when front-end MTA
    $final_spam_destiny       = D_BOUNCE;
    $final_bad_header_destiny = D_PASS;     # False-positive prone (for spam)
    ...

Finally, edit ``/etc/amavis/conf.d/50-user`` and add the line ``$pax='pax';`` in the middle:

::

    use strict;
    
    #
    # Place your configuration directives here.  They will override those in
    # earlier files.
    #
    # See /usr/share/doc/amavisd-new/ for documentation and examples of
    # the directives you can use in this file
    #
    
    $pax='pax';
    
    #------------ Do not modify anything below this line -------------
    1;  # ensure a defined return

Afterwards, run these commands to add the clamav user to the amavis group and to restart amavisd-new and ClamAV:

::

    $ sudo adduser clamav amavis
    $ sudo /etc/init.d/amavis restart
    $ sudo /etc/init.d/clamav-daemon restart
    $ sudo /etc/init.d/clamav-freshclam restart


.. tip ::

    You might get this error, I don't know how to fix it!

    ::

        $ sudo /etc/init.d/clamav-daemon restart
        Stopping ClamAV daemon: clamd.
        Starting ClamAV daemon: clamd LibClamAV Warning: ***********************************************************
        LibClamAV Warning: ***  This version of the ClamAV engine is outdated.     ***
        LibClamAV Warning: *** DON'T PANIC! Read http://www.clamav.net/support/faq ***
        LibClamAV Warning: ***********************************************************
        .

    I think it will go away when the Debian team get to packaging the latest version in the volatile repository.


Now we have to configure Postfix to pipe incoming email through ``amavisd-new``. Emails arrive to Postfix on port 25, it will send them to AMaViS on port 10024 (XXX and receive them on port 10025???)

::

    $ sudo postconf -e 'content_filter = amavis:[127.0.0.1]:10024'
    $ sudo postconf -e 'receive_override_options = no_address_mappings'

Afterwards append the following lines to ``/etc/postfix/master.cf``:

::

    ...
    amavis unix - - - - 2 smtp
            -o smtp_data_done_timeout=1200
            -o smtp_send_xforward_command=yes
    
    127.0.0.1:10025 inet n - - - - smtpd
            -o content_filter=
            -o local_recipient_maps=
            -o relay_recipient_maps=
            -o smtpd_restriction_classes=
            -o smtpd_client_restrictions=
            -o smtpd_helo_restrictions=
            -o smtpd_sender_restrictions=
            -o smtpd_recipient_restrictions=permit_mynetworks,reject
            -o mynetworks=127.0.0.0/8
            -o strict_rfc821_envelopes=yes
            -o receive_override_options=no_unknown_recipient_checks,no_header_body_checks
            -o smtpd_bind_address=127.0.0.1

Then restart Postfix:

::

    $ sudo /etc/init.d/postfix restart

Checking the Setup
------------------

Now run:

::

    $ sudo netstat -tap | grep master
    tcp        0      0 *:smtp                  *:*                     LISTEN      4959/master     
    tcp        0      0 localhost.localdo:10025 *:*                     LISTEN      4959/master    
    $ sudo netstat -tap | grep amavis
    tcp        0      0 localhost.localdo:10024 *:*                     LISTEN      3281/amavisd (maste

and you should see Postfix (master) listening on port 25 (smtp) and 10025, and amavisd-new on port 10024:


.. tip ::

    If you don't see the amavis server running check syslog:

    ::

        sudo tail -f /var/log/syslog

    If you see a line like this:

    ::

        (!)Net::Server: 2009/05/18-18:38:38 Couldn't fork: [Cannot allocate memory]\n\n  at line 293 in file /usr/share/perl5/Net/Server.pm

    it means you don't have enough free memory. Run ``free -m`` and look at the second line, ``free`` column to see how much memory is free. 

    If you are running OpenVZ, run:

    ::

        $ sudo cat /proc/user_beancounters

    If the right-most value for ``privvmpages`` is not 0 you need to allocate more privvmpages to the VE.

    ::

        # vzctl set 2 --privvmpages  128536 --save

    Now restart the VE.

Install Razor, Pyzor And Configure SpamAssassin
===============================================

Razor and Pyzor are spamfilters that use a collaborative filtering network. To install Razor and Pyzor, run

::
 
    $ sudo apt-get install razor pyzor

Now we have to tell SpamAssassin to use these three programs. Edit
``/etc/spamassassin/local.cf`` and add the following lines to it at the end:

::

    ...
    #pyzor
    use_pyzor 1
    pyzor_path /usr/bin/pyzor
    
    #razor
    use_razor2 1
    razor_config /etc/razor/razor-agent.conf
    
    #bayes
    use_bayes 1
    use_bayes_rules 1
    bayes_auto_learn 1

You can check your SpamAssassin configuration by executing:

::

    $ spamassassin --lint

It shouldn't show any errors.

Restart ``amavisd-new`` afterwards:

::

    $ sudo /etc/init.d/amavis restart

Now we update our SpamAssassin rulesets as follows:

::

    $ sudo sa-update --no-gpg

This can take a minute or two.

Next we create a cron job so that the rulesets will be updated regularly. Run

::

    $ sudo crontab -e

to open the cron job editor. Create the following cron job:

::

    0 3 */2 * * /usr/bin/sa-update --no-gpg &> /dev/null

This will update the rulesets every second day at 3am from now on.  

Testing the Setup
=================

With everything set up, try sending a test email through the system.

