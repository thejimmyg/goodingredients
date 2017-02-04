Setting up Apache
+++++++++++++++++

Variables: pylonsbook-live, PylonsBookSite, pylonsbooksite, ve2.fourier.example.com, 010, pylonsbook-live.pylonsbook.com, apps.example.com, 192.168.100.2

On a newly created virtual environment using Nginx on the proxy you will need
to follow the following steps to get Apache installed correctly and proxied to
correctly from PLPA.

Install
=======

First update your packages:

::

    sudo apt-get update

Now install Apache and mod_wsgi (the adaptor that allows embedding of Python)
and ``elinks`` so that you can test Apache locally:

::

    sudo apt-get install libapache2-mod-wsgi elinks

It's also a good idea to install your editor of choice. I install ``vim-nox``
becuase the default ``vim`` install misses out quite a few standard vim
features:

::

    sudo apt-get install vim-nox

I find I need to restart Apache before it starts serving:

::

    sudo /etc/init.d/apache2 restart

Now run elinks to test the installation:

::

    elinks http://localhost:80/

By default if Apache is running but the host name doesn't match the ``ServerName`` or ``ServerAlias`` directives in any of the virtual host configurations, Apache serves the content from the first virtual host. Since the only virtual host configured is the ``default`` site you should see Apache's default page saying ``It works!``. 

Press ``q`` to quit.
 
.. tip ::

   If you have problems you can install telnet:

   ::

       sudo apt-get install telnet

   Then test the connection manually by typing ``GET /`` the enter. If nothing
   happend press ``Ctrl+]`` and enter then type ``quit`` at the ``telent>`` prompt
   to exit. Otherwise you should see this:

   ::

       $ telnet 127.0.0.1 80
       Trying 127.0.0.1...
       Connected to 127.0.0.1.
       Escape character is '^]'.
       GET /
       <html><body><h1>It works!</h1></body></html>
       Connection closed by foreign host.

Configuring the It works! Behaviour
===================================

By default if Apache is running but the host name doesn't match the
``ServerName`` or ``ServerAlias`` directives in any of the virtual host
configurations, Apache serves the content from the first virtual host. By
default Apache always names the ``default`` site ``000-default`` when it is
enables to ensure the configuration always comes first. Since the default site
is already enables and always comes first, we can customise the "It works!"
message to say "The Apache webserver could not find the requested website. This
is likely to be a temporary problem but if it persists please contact support
to let us know about the problem." You can then have "contact support" linked
to your commany's support page. 

By making this change you'll always know when DNS and Nginx configurations are
forwarding correctly to Apache and your users will know to report the problem
if they ever see it on a production site.

Edit ``/var/www/index.html`` to adjust the content.

::

    sudo vim /var/www/index.html

Apache is now ready for you to configure vitual hosts but first you need to
have a brief idea of a typical deployment process.

The Deployment Process
======================

The way we are going to work is to create a new user for each domain (or
related set of domains) which we want to work with. Generally speaking
deployment looks like this:

* Develop on the development instance (your laptop for example)
* When it is ready deploy it on the test instance to run the tests (on 
  a server configured identically to the staging and live servers)
* If the tests all pass, deploy to staging for testing by the users
* If the users are happy deploy to live

If you want to have less servers you can actually handle the test, staging and
live deployments on the same server but create different user accounts for each
case. For example if we were setting up the pylonsbook.com domain you could deploy
one copy as the ``pylonsbook-dev`` user, one copy as the ``pylonsbook-test`` user,
one as ``pylonsbook-live`` and one as ``pylonsbook-live``. In fact, it is
recommened you use this naming convention anyway, even if you use separate
servers because it means that you'll never be confused as to which instance you
are working on at a particular time.

Create User
===========

With the deployment process in mind, let's imagine we are setting up the
staging instance. We'll therefore need to create an ``pylonsbook-live`` user.

.. tip ::

   Technically speaking you don't need separate users for each virtual host,
   and in fact you can manage everything from the ``/var/www`` directory, but
   following these instructions and creating a new user each time is a much more
   sensible idea; you are on your own otherwise!

Create the user like this:

::

    $ sudo adduser pylonsbook-live

Enter the new password twice then accept the default for everything and choose
``y`` at the end to create the user.

This creates the ``/home/pylonsbook-live`` directory where you should keep all
code related to this deployment.

Granting Permissions
====================

Permissions frequently cause problems when lots of different users are working
with different deployments. In this case sudo can come to the rescue.

In our set up we are going to create a set of users, all of whom have
permission to stop, start and restart Apache and to run apt-get commands
including ``update``, ``upgrade`` and ``install``. They'll also be able to run
commands as any of the users set up for each site such as ``pylonsbook-live``.
All commands run with sudo will be logged so that if there are problems the
sysadmin can see which user caused them. This is necessary because all the
different users will be running commands as the ``pylonsbook-live`` user so the
usual permissions checks won't be much good.

.. note ::

   The alternative approach of setting the group read, write and execute flags
   on all files in the ``/home/pylonsbook-live`` directory and then adding other
   users to the pylonsbook-live group isn't quite such a good solution because it
   is often useful to give group write permissions to the Apache user
   (``www-data``) instead.

To set this up run ``visudo`` and update it so that it looks like this.

::

    # /etc/sudoers
    #
    # This file MUST be edited with the 'visudo' command as root.
    #
    # See the man page for details on how to write a sudoers file.
    #
    
    Defaults env_reset, logfile=/var/log/sudolog
    
    # Host alias specification
    
    # User alias specification
    User_Alias WEB_DEPLOYERS = pylonsbook-live
    
    
    # Cmnd alias specification
    Cmnd_Alias  WEB_DEPLOYMENT_COMMANDS  = /usr/sbin/a2ensite, /usr/sbin/a2dissite, /usr/sbin/a2enmod, /usr/sbin/a2dismod, /etc/init.d/apache2
    
    # User privilege specification
    root    ALL=(ALL) ALL
    
    # Uncomment to allow members of group sudo to not need a password
    # (Note that later entries override this, so you might need to move
    # it further down)
    # %sudo ALL=NOPASSWD: ALL
    james ALL=(ALL)ALL
    WEB_DEPLOYERS ALL = (ALL) WEB_DEPLOYMENT_COMMANDS

For each new web deployer you add, update ``WEB_DEPLOYERS`` and they will have
permission to run the ``WEB_DEPLOYMENT_COMMANDS`` as any user.

For more information see:

* http://www.gentoo.org/doc/en/sudo-guide.xml
* http://aplawrence.com/Basics/sudo.html

Deploying the Code
==================

Virtually all sites will require some sort of dynamic content so I'll show you
how to set up Apache with WSGI support.

Become the ``pylonsbook-live`` user and change directory to the user's home
directory:

::

    cd /home/pylonsbook-live/ 

If you are using the Flows framework you are likely to want to track your
deployment code as part of the main codebase which means your Apache virtual
host configuration will be
``/home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/vhost.conf``.
If you want to put it elsewhere that's fine, you'll just have to update the
path in the rest of this tutorial.

First set up the basic structure and the virtual Python environment:

::

    sudo -u pylonsbook-live mkdir download lib log
    cd download
    sudo -u pylonsbook-live wget http://pylonsbook.com/virtualenv.py
    sudo -u pylonsbook-live python virtualenv.py ../env
    cd ../

Then create the directory for the virtual host and the static files (if you are
actually using a Flows application you could just clone the code into the
repository at this point):

::

    sudo -u pylonsbook-live mkdir -p lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/
    sudo -u pylonsbook-live mkdir -p lib/PylonsBookSite/code/trunk/pylonsbooksite/static

Create an ``index.html`` file too:

::

    sudo -u pylonsbook-live vim lib/PylonsBookSite/code/trunk/pylonsbooksite/static/index.html

Add this content:

::

    <html><head><title>Static Files Work</title></head><body><h1>Static Files Work</h1></body></html>

You should now have these three directories:

::

    $ ls
    download  env  lib  log
 
Now you can create the virtual host. Create the file:

::

    sudo -u pylonsbook-live vim /home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/vhost.conf

with this content:

::

    <VirtualHost *:80>
        ServerName pylonsbook-live.pylonsbook.com

        # Logs
        ErrorLog /home/pylonsbook-live/log/apache.error.log
        CustomLog /home/pylonsbook-live/log/apache.access.log common
    
        # Static Files
        DocumentRoot /home/pylonsbook-live/lib/PylonsBookSite/code/trunk/pylonsbooksite/static
        <Directory /home/pylonsbook-live/lib/PylonsBookSite/code/trunk/pylonsbooksite/static>
            Options FollowSymlinks
            DirectoryIndex index.html
            AllowOverride None
            Order allow,deny
            Allow from all
        </Directory>

        # Dynamic Files
        WSGIScriptAlias /wsgi /home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/dispatch.wsgi
        <Location />
        # Redirect all requests not available on the filesystem to /wsgi
        RewriteBase /
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule ^(.*)$ wsgi/$1 [L,QSA]
        </Location>

        # Error Documents
        ErrorDocument  400  /errordocument/400.html
        ErrorDocument  401  /errordocument/401.html
        ErrorDocument  403  /errordocument/403.html
        ErrorDocument  404  /errordocument/404.html
        ErrorDocument  500  /errordocument/500.html
    </VirtualHost>

You'll need to create a symbolic link to tell Apache that the site exists
because we have not put the virtual host in the standard location
(``/etc/apache2/sites-available``):

::

    sudo ln -s /home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/vhost.conf /etc/apache2/sites-available/pylonsbook-live.pylonsbook.com

In this case the PylonsBook staging site will be hosted at pylonsbook-live.pylonsbook.com
so this is what I name the symlink. Apache loads the sites in alphabetical
order. Most of the time the order of the virtual hosts doesn't matter but you
can manually control the order by putting a number in front of the domain as
shown above.

Let's now enable mod_rewrite which we used in the virutal host:

::

    $ sudo a2enmod rewrite
    Enabling module rewrite.
    Run '/etc/init.d/apache2 restart' to activate new configuration!

Now create the WSGI script:

::

   sudo -u pylonsbook-live vim /home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/dispatch.wsgi

The public one looks like this:

::

    #!/home/pylonsbook-live/env/bin/python

    def application(environ, start_response):
        # Protect against people accessing /wsgi directly
        if environ['REQUEST_URI'] == '/wsgi' or environ['REQUEST_URI'].startswith('/wsgi/'):
            start_response('404 Not found', [('Content-Type', 'text/plain')])
            return ['Not found.']
        start_response('200 OK', [('Content-Type', 'text/html')])
        return ['Hello world!<br /> Environ: '+str('<br />'.join([str([k, v]) for k,v in environ.items()]))]

Here's a similar version for accessing the Python environement set up in
``~/env`` and for deploying a flows app looks like this:

::

    #!/home/pylonsbook-live/env/bin/python

    import os
    import site
    site.addsitedir('/home/pylonsbook-live/env/lib/python2.5/site-packages')
    
    from pylonsbooksite.frmework.dispatch import on_load_app
    from pylonsbooksite.framework.config import on_load_config
    from configconvert import parse_config
    
    config_file = '/home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/app.conf'
    
    options = parse_config(config_file)
    config = on_load_config(
        options=options,
        package=options['app.package'],
        config_file=config_file,
    )
    flows_app = on_load_app(config)

    def application(environ, start_response):
        # Protect against people accessing /wsgi directly
        if environ['REQUEST_URI'] == '/wsgi' or environ['REQUEST_URI'].startswith('/wsgi/'):
            start_response('404 Not found', [('Content-Type', 'text/plain')])
            return ['Not found.']
        return flows_app(environ, start_response)

Now install the application and its dependencies:

::

    cd /home/pylonsbook-live/lib/PylonsBookSite/code/trunk
    ~/env/bin/python setup.py develop

Change the permissions to make it executable and check you can run it:

::

    cd /home/pylonsbook-live/lib/PylonsBookSite/deploy/ve2.fourier.example.com/pylonsbook-live/
    chmod 755 dispatch.wsgi
    sudo -u pylonsbook-live /home/pylonsbook-live/env/bin/python dispatch.wsgi

Now you can enable the virtual host:

::

    sudo a2ensite 010-pylonsbook-live.pylonsbook.com

The virtual hosts will be loaded in alphabetical order so you can replace the
``010`` part with any number you prefer but avoid using ``000`` as this might
interfere with the ``default`` virtual host you set up earlier.

Then reload Apache:

::

    sudo /etc/init.d/apache2 reload

Create the Error Documents
==========================

You need to create the 5 error documents: 400.html, 401.html, 403.html, 404.html, 500.html in the ``/home/pylonsbook-live/lib/PylonsBookSite/code/trunk/pylonsbooksite/static/errordocument`` directory. Here's an example for the 401 error:

::

    <html>
    <head><title>Mercurial Repositories</title></head>
    <body>
    <h1>Authentication Required/h1>
    
    <p>This server could not verify that you are authorized to access the
    document requested. Either you supplied the wrong credentials (e.g., bad
    password), or your browser doesn't understand how to supply the credentials
    required.</p>
    
    </body>
    </html>

Testing
=======

You can't test it directly yet because we haven't set up Nginx to proxy to it
but you can use the command line browser like elinks to check it is working.
First temporarily add the hostname to ``/etc/hosts`` by chaning this line:

::

    127.0.0.1 localhost.localdomain localhost

to:

::

    127.0.0.1 localhost.localdomain localhost pylonsbook-live.pylonsbook.com

Then start elinks:

::

    elinks http://pylonsbook-live.pylonsbook.com

You should see the message ``Static Files Work`` (or the equivalent from your
application). Press ``q`` to exit. If there is a problem look at the logs for hints:

::

    sudo tail -f /var/log/apache2/error.log
    tail -f /home/pylonsbook-live/log/apache.error.log


Now try a different URL and you should find that it gets redirected to the WSGI
app. Apache calls the application defined in ``flows.wsgi`` which in turn loads
the dynamic pages from the application.

Set up Nginx
============

Next you need to setup Nginx on the other machine so that
http://pylonsbook-live.pylonsbook.com is proxied to the Apache server and so
that http is redirected to https.

Create the file ``/etc/nginx/sites-available/pylonsbook-live.pylonsbook.com``
with these contents:

::

    server {
        listen   80;
        server_name  www.pylonsbook-live.pylonsbook.com;
        rewrite ^/(.*) http://pylonsbook-live.pylonsbook.com/$1 permanent;
    }

    server {
        listen   80;
        server_name pylonsbook-live.pylonsbook.com;
    
        location / {
            access_log off;
            proxy_pass http://192.168.100.3:80;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

Now enable the site:

::

    sudo ln -s /etc/nginx/sites-available/pylonsbook-live.pylonsbook.com /etc/nginx/sites-enabled/pylonsbook-live.pylonsbook.com

Disable the existing configuration which we no longer need if you haven't already done so:

::

    sudo rm /etc/nginx/sites-available/default

Test the configuration:

::

    sudo nginx -t -c /etc/nginx/nginx.conf
    2009/06/12 13:40:20 [info] 894#0: the configuration file /etc/nginx/nginx.conf syntax is ok
    2009/06/12 13:40:20 [info] 894#0: the configuration file /etc/nginx/nginx.conf was tested successfully

Reload Nginx by obtaining its PID and sending it an HUP signal:

::

    ps aux | grep nginx
    root       301  0.0  0.1  27624   896 ?        Ss   Jun10   0:00 nginx: master process /usr/sbin/nginx
    sudo kill -HUP 27624

Now check the existing sites are still up and running correctly.

Modify an Existing BIND Domain
==============================

If you already have the files for the domain set up, adding a new subdomain is
very easy. In this case edit ``/etc/bind/db.pylonsbook.com`` and add a new A
record which points to the IP address of the server serving Nginx. You'll also
need to update the serial number with the current date. If the date is the
same, increment the number after it instead (``01`` in this case). The serial
number must change for the updated record to take full effect.

Here's an example:

::

    $TTL 84924 ; zone default = 1 day or 84924 seconds
    $ORIGIN pylonsbook.com.
    pylonsbook.com.    IN      SOA   ns0.apps.example.com. apps.example.com. (
        2009062901 ; serial number
        3h         ; refresh =  3 hours
        15M        ; update retry = 15 minutes
        15M        ; expiry = 3 weeks + 12 hours
        15M        ; minimum = 2 hours + 20 minutes
    )
    pylonsbook.com.       84924  IN  NS  ns0.apps.example.com.
    pylonsbook.com.       84924  IN  NS  ns1.apps.example.com.
    pylonsbook-live.pylonsbook.com.   84924  IN  A   188.40.40.171

Now reload BIND:

::

    $ sudo /etc/init.d/bind9 reload
    Reloading domain name service...: bind9.

Check there weren't any error messages:

::

    sudo tail -f /var/log/syslog

If you've made a mistake and correct it remember to update the serial number, otherwise the change won't take effect when you try to reload bind again.

Adding a new Domain to BIND
===========================

If you are setting up a new domain and cannot simply modify an existing set of
records you'll need to take some extra steps.

First of all you'll need to create the zone file for the domain. By convention the filename should be the same as the domain the zone is for but start with ``db.``:

::

    sudo vim /etc/bind/db.pylonsbook.com

Here's some sample content you can as a template:

::

    $TTL 84924 ; zone default = 1 day or 84924 seconds
    $ORIGIN pylonsbook.com.
    pylonsbook.com.    IN      SOA   ns0.apps.example.com. apps.example.com. (
        2009092302 ; serial number
        3h         ; refresh =  3 hours
        15M        ; update retry = 15 minutes
        15M        ; expiry = 3 weeks + 12 hours
        15M        ; minimum = 2 hours + 20 minutes
    )
    pylonsbook.com.       84924  IN  NS  ns0.apps.example.com.
    pylonsbook.com.       84924  IN  MX  10  plpa.pylonsbook.com.
    www.pylonsbook.com.   84924  IN  A   188.40.40.171
    pylonsbook.com.       84924  IN  A   188.40.40.171

This sets up a zone with a single mail server, a single nameserver and two A
records. Every time you change the zone you *must* update the serial number. A
good convention is to always use today's date followed by the digits ``01`` and
then incrment the last two digits every time you change the zone more than once
in the same day so that the serial number is incremented on every change.

Next you need to tell BIND to use this new zone. Edit:

::

    sudo vim /etc/bind/named.conf.local

and add this content after the existing definitions:

::

    zone "pylonsbook.com" {
        type master;
        file "/etc/bind/db.pylonsbook.com";
    };

Now you can reload bind:

::

    $ sudo /etc/init.d/bind9 reload
    Reloading domain name service...: bind9.

Check there weren't any error messages:

::

    sudo tail -f /var/log/syslog

If you've made a mistake and correct it remember to update the serial number, otherwise the change won't take effect when you try to reload bind again.


Testing
=======

You can test the DNS settings are correct like this:

::

    dig @ns0.apps.example.com pylonsbook-live.apps.example.com A

If everything is working you should see this in the results:

::

    ;; ANSWER SECTION:
    pylonsbook-live.apps.example.com.	84924	IN	A	188.40.40.171

At this point if you visit http://pylonsbook-live.pylonsbook.com from any
browser you should see your new site.

Finally you should remove the entry you added during testing from the
``/etc/hosts`` file on the server with Apache so that the BIND server becomes
the definitive authority.

