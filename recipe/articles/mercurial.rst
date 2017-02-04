Setting up Mercurial
====================

We're going to have multiple copies of the Mercurial server running so that we
can easily group related repositories. This will allow us to keep private
repositories from appearing alongside public ones.

Since we are using Nginx, we'll have it decode the SSL and proxy to Apache over
HTTP. This means that we don't need to set up Apache with an SSL certificate.

Configuring Apache
------------------

Install some required software:

::

    $ sudo apt-get install libapache2-mod-wsgi mercurial libapache2-mod-
    $ sudo a2enmod auth_digest
    $ sudo /etc/init.d/apache2 start

Start by creating the virtual host. Save this as ``/etc/apache2/hg.example.com``:

::
    
    # Redirect virtual hosts
    <VirtualHost *:80>
        ServerName hg.example.com
        DocumentRoot /var/www/vhost/hg.example.com/htdocs
        ErrorLog /var/log/apache2/hg.example.com-error_log
        CustomLog /var/log/apache2/hg.example.com-access_log common
    
        WSGIScriptAlias /public /var/www/vhost/hg.example.com/cgi-bin/hgwebdir_public.wsgi
        WSGIScriptAlias /private /var/www/vhost/hg.example.com/cgi-bin/hgwebdir_private.wsgi
    
        <Directory /var/www/vhost/hg.example.com/htdocs>
            Options FollowSymlinks
            DirectoryIndex index.html
    
            AllowOverride None
            Order allow,deny
            Allow from all
        </Directory>
    
        # Require permission for writing to any repository
        <Location / >
            Order allow,deny
            Allow from all
    
            AuthType Digest
            AuthName "Private Repositories"
            AuthDigestDomain /
            AuthDigestProvider file
            AuthUserFile /var/www/vhost/hg.example.com/user/hg.digest.passwd
    
            <LimitExcept GET>
                 Require valid-user
            </LimitExcept>
        </Location>

        # Don't allow reads unless someone is signed in
        <Location /private>
            Order allow,deny
            Allow from all
        
            AuthType Digest
            AuthName "Private Repositories"
            AuthDigestDomain /
            AuthDigestProvider file
            AuthUserFile /var/www/vhost/hg.example.com/user/hg.digest.passwd
        
            <Limit GET POST DELETE PUT>
               Require valid-user
            </Limit>
        </Location>
   
        <Directory /var/www/vhost/hg.example.com/htdocs/errordocs>
         AllowOverride none
         Options MultiViews IncludesNoExec FollowSymLinks
         AddType text/html .shtml
         AddHandler server-parsed .shtml
        </Directory>
        ErrorDocument  400  /errordocs/400.html
        ErrorDocument  401  /errordocs/401.html
        ErrorDocument  403  /errordocs/403.html
        ErrorDocument  404  /errordocs/404.html
        ErrorDocument  500  /errordocs/500.html
    </VirtualHost>

Now disable the default site:

::

    sudo a2dissite default

Now create the required directories:

::

    sudo mkdir /var/www/vhost
    sudo mkdir /var/www/vhost/hg.example.com
    cd /var/www/vhost/hg.example.com
    sudo mkdir cgi-bin htdocs conf user repo
    sudo mkdir htdocs/errordocs
    sudo mkdir repo/public
    sudo mkdir repo/private

Now create the two WSGI scripts. The public one looks like this and should be
saved as ``/var/www/vhost/hg.example.com/cgi-bin/hgwebdir_public.wsgi``:

::

    from mercurial import demandimport; demandimport.enable()
    from mercurial.hgweb.hgwebdir_mod import hgwebdir
    app = hgwebdir('/var/www/vhost/hg.example.com/conf/public.config')

    def application(environ, start_response):
        # Trick mercurial into thinking we are using https so that it
        # doesn't complain about 'ssl required' when you push
        environ['wsgi.url_scheme'] = 'https'
        return app(environ, start_response)

The private one looks like this and should be saved as
``/var/www/vhost/hg.example.com/cgi-bin/hgwebdir_private.wsgi``:

::

    from mercurial import demandimport; demandimport.enable()
    from mercurial.hgweb.hgwebdir_mod import hgwebdir
    app = hgwebdir('/var/www/vhost/hg.example.com/conf/private.config')

    def application(environ, start_response):
        # Trick mercurial into thinking we are using https so that it
        # doesn't complain about 'ssl required' when you push
        environ['wsgi.url_scheme'] = 'https'
        return app(environ, start_response)

Now create ``/var/www/vhost/hg.example.com/conf/public.config`` like this:

::

    [web]
    allow_push = *
    push_ssl = false
    style = gitweb
    allow_archive = bz2 gz zip
    contact = James Gardner, james at example.com
    
    [collections]
    /var/www/vhost/hg.example.com/repo/public = /var/www/vhost/hg.example.com/repo/public
    
    [hooks]
    changegroup =
    # reload wsgi application
    changegroup.mod_wsgi = touch /var/www/vhost/hg.example.com/cgi-bin/hgwebdir_public.wsgi

Create ``/var/www/vhost/hg.example.com/conf/private.config`` like this:

::

    [web]
    allow_push = *
    push_ssl = false
    style = gitweb
    allow_archive = bz2 gz zip
    contact = James Gardner, james at example.com
    
    [collections]
    /var/www/vhost/hg.example.com/repo/private = /var/www/vhost/hg.example.com/repo/private
    
    [hooks]
    changegroup =
    # reload wsgi application
    changegroup.mod_wsgi = touch /var/www/vhost/hg.example.com/cgi-bin/hgwebdir_private.wsgi

Now create a user by creating ``/var/www/vhost/hg.example.com/user/hg.digest.passwd`` like this:

::

    sudo htdigest -c /var/www/vhost/hg.example.com/user/hg.digest.passwd "Private Repositories" username

Finally add an index page. Here's a very simple example you can save as ``/var/www/vhost/hg.example.com/htdocs/index.html``:

::

    <html>
    <head><title>Mercurial Repositories</title></head>
    <body>
    <h1>Mercurial Repositories</h1>
    <ul>
      <li><a href="/public/">Public Repositories</a></li>
      <li><a href="/private/">Private Repositories</a></li>
    </ul>
    </body>
    </html>

Now you can enable the virtual host:

::

    sudo a2ensite hg.example.com

Then reload Apache:

::

    sudo /etc/init.d/apache2 reload

Create the Error Documents
--------------------------

You need to create the 5 error documents: ``400.html``,
``401.html``, ``403.html``, ``404.html``,
``500.html`` in the ``/var/www/vhost/hg.example.com/htdocs/errordocs``
directory. Here's an example for the 401 error:

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
-------

You can't test it directly yet because we haven't set up Nginx to proxy to it but you can use a command line browser like ``elinks`` to check it is working. First add the hostname to ``/etc/hosts`` by chaning this line:

::

    127.0.0.1 localhost.localdomain localhost

to

::

    127.0.0.1 localhost.localdomain localhost hg.example.com

Then start elinks:

::

    elinks hg.example.com

See also: http://www.selenic.com/mercurial/wiki/modwsgi

Set up Nginx
------------

Finally you need to setup Nginx on the other machine so that https://hg.example.com is proxied to the
Apache server and so that http is redirected to https.

We need to install some SSL tools:

::

    sudo apt-get install openssl

First create the directories:

::

    sudo mkdir /etc/nginx/ssl
    sudo mkdir /etc/nginx/ssl/hg.example.com
    
Then create the certificate ensuring that the common name you enter matches the domain name, eg hg.example.com.

Create SSL Certificates
-----------------------

To generate the Certificate Signing Request you should create your own key. You
can run the following command from a terminal prompt to create the key:

::

    openssl genrsa -des3 -out server.key 1024

You will need to install the packages for ``openssl`` if they are not already
installed. Using the command above will create a key which uses a password. The
minimum length when specifying ``-des3`` is four characters. It should include
numbers and/or punctuation and not be a word in a dictionary. Also remember
that your passphrase is case-sensitive.

Re-type the passphrase to verify. Once you have re-typed it correctly, the
server key is generated and stored in ``server.key`` file. Using a key with a
passphrase like this can be inconvenient because every time you restart Nginx
the passphrase has to be manually entered so if you server were to be rebooted,
Nginx would not start without manual intervention.

You can also run your secure web server without a passphrase but it is highly
insecure and a compromise of the key means a compromise of the server as well.
You can generate a key without a passphrease by leaving out the ``-des3`` switch in
the generation phase or by issuing the following command at a terminal prompt
to convert your existing key.

::

    openssl rsa -in server.key -out server.key.insecure

Once you have the key (with the passphrase or without) you need to generate the
certificate signing request.

::

    openssl req -new -key server.key -out server.csr

You will be prompted for Company Name, Site Name, Email ID, etc The details you
enter here will form part of the certificate. It is important that the Common
Name you enter (CN) matches the domain name the secure certificate is for
otherwise the certificate won't work.

Now you have your certificate signing request you can either generate a
self-signed certificate or purchase a signed certificate from a Certificate
Authority. If you choose to sign the certificate yourself your users will be
prompted by their web browser each time they visit the site because the browser
will not recognise the certificate. As a rule production sites should not use
self-signed certificates.

To sign the certificate yourself issue this command:

::

    openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

We will use the key and certificate when setting up Apache.


Debugging
---------

If you are using Firefox and simply get an Error message alert when you try to
use the HTTPS version of your site it is likely you have made an error in
creating your certificate. Check that the Common Name you specified when
creating the certificate signing request matches the domain name you are using.

If you need to perform further debugging these commands might be useful: 

::

    netstat -ta
    openssl s_client -connect localhost:443 -state -debug

You should also check the main Apache error logs for clues as well as your
virtual host's error log:

::

    tail /var/log/apache2/error.log




Configuring
-----------

Now add this configuration to ``/etc/nginx/sites-available/hg.example.com``:

::

    server {
        listen   80;
        server_name  hg.example.com;
        rewrite ^/(.*) https://hg.example.com/$1 permanent;
    }

    server {
        # Important to have the public IP in the listen directive so that 
        # you can host multiple https domains, one per public IP.
        listen               188.40.40.171:443;
        ssl                  on;
        ssl_certificate      /etc/nginx/ssl/hg.example.com/host.pem;
        ssl_certificate_key  /etc/nginx/ssl/hg.example.com/host.key;
        keepalive_timeout    70;
        # Change this to be the maximum size of changeset you want to allow to be pushed.
        client_max_body_size 300m;
        access_log  /var/log/nginx/localhost.access.log;

        location / {
            access_log off;
            proxy_pass http://192.168.100.2:80;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

.. tip ::

    You may find you need to change the size of the ``client_max_body_size``
    depending on the maximum size you expect an ``hg push`` command will send.

    If you set this too low you'll see the error ``abort: error: Broken pipe``.
    You can re-run the ``hg push`` with the ``-v`` and ``--debug`` options to debug
    the error:

    ::

        hg push -v --debug
        using https://hg.example.com/private/JimmygSite
        pushing to https://hg.example.com/private/JimmygSite
        sending capabilities command
        http authorization required
        realm: Private Repositories
        user: james
        password: 
        capabilities: unbundle=HG10GZ,HG10BZ,HG10UN lookup changegroupsubset
        sending heads command
        searching for changes
        common changesets up to e3adeb1c87e8
        1 changesets found
        List of changesets:
        fc10366d7f2b8bbf9ec092282df2590a396f0701
        sending unbundle command
        sending 33583845 bytes
        abort: error: Broken pipe

    As you can see, the amount sent is 33583845 bytes, which was over the 30MB
    limit this particular server was configured for (which is why the example
    configuration uses 300MB). In the Nginx logs you'd see something like this:

    ::
                                                                                                                                   
        2009/06/13 22:05:36 [error] 1246#0: *354 client intended to send too large body: 33583845 bytes, client: 84.9.40.119, server: plpa.example.com, request: "POST /private/JimmygSite?cmd=unbundle&heads=e3adeb1c87e8965906b64b2c909870d5894b6efb HTTP/1.1", host: "hg.example.com"

Now enable the site:

::

    sudo ln -s /etc/nginx/sites-available/hg.example.com /etc/nginx/sites-enabled/hg.example.com

Disable the existing configuration which we no longer need:

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

Using a GoDaddy Certificate
---------------------------

Create the certificate signing request:
 
::

    openssl genrsa -out godaddy.key 1024
    openssl req -new -key godaddy.key -out godaddy.csr

The process of buying SSL is quite involved:

1. You pay for the SSL order first, and you get 1 ssl credit in Godaddy account.
2. You configure the credit, and submit the whole body text from mysite.csr, making sure you choose Apache as the web server.
3. Godaddy send you an approval email request for domain owner (should be you). (very fast).
4. you click email link and approve, then Godaddy email you the actual ssl certificate files. (takes 5 mins)

Make sure the email address you give in the csr upload page actually works.
Also, if you don't get the email you can click the "Re-issue" button which
allows you to download it directly.

::

    sudo apt-get install unzip
    $ sudo unzip ~/certbundle.zip 

Then you need to combine the certificate with the intermediate file:

::

    cat hg.example.com.crt gd_bundle.crt > combined.crt

The ``combined.crt`` file can then be used in place of the ``host.pem``
certificate you generated yourself earlier.

See also: http://nginx.groups.wuyasea.com/articles/how-to-setup-godaddy-ssl-certificate-on-nginx/2

Adding a Repository
-------------------

Adding a new repository is easy, you simply create a normal Mercurial
repository in either the public or the private directory, depending on which
you prefer.

For example to create a public directory:

::

    cd /var/www/vhost/hg.example.com/repo/public
    sudo mkdir new_repo
    cd new_repo
    sudo hg init
    touch README.txt
    hg add README.txt
    hg ci -m "First commit"
    cd ../
    chown -R www-data:www-data /var/www/vhost/hg.example.com/repo
    sudo /etc/init.d/apache2 restart

You can then use it like this:

::

    $ hg clone https://hg.example.com/public/new_repo 

Before you can check in changes remotely though you need to add the user to the ``.hg/hgrc`` file. As shown in the next section.

Adding extra Users
------------------

If you have already created
``/var/www/vhost/hg.example.com/user/hg.digest.passwd`` you can add a new user
like this:

::

    htdigest /var/www/vhost/hg.example.com/user/hg.digest.passwd "Private Repositories" username

Either way you will be asked for a password for the user ``username``.

Once the user has been created you need to edit the ``.hg/hgrc`` file in each
repository you want that user to have access to. For example in the file below,
users with the usernames ``james`` and ``stephen`` are allowed to push
changes to the repository. We've also set some other information.

::

    [web]
    allow_push = james, stephen
    style = gitweb
    allow_archive = bz2 gz zip
    description = Build forms quickly and easily using groups of simple helper functions.
    contact = James Gardner
    push_ssl = False

.. tip ::

    If you are using this recipe as a basis for creating lots of private
    repositories, it is a good idea to use the same realm so that users with the
    same usrename who sign into one part of the site they have access to don't have
    to sign in again when they access a different private repository they have
    access to where their username and password are the same.

Moving Repositories
-------------------

If you are moving a repositry from a different location you can simply move the
folder. Just update the ``.hg/hgrc`` file so that it has the correct default
path for the new location. eg:
 
::

    [paths]
    default = https://hg.example.com/public/FormBuild

Converting From Subversion
--------------------------

If you have an old subversion repository and you want to convert it to
mercurial you need to install the Python SWIG bindings for subversion:

::

    sudo apt-get install python-subversion

.. tip ::

   The mercurial instructions suggest you also explicitly need to enable the ``hg
   convert`` extensions but actually you don't.

You can then convert the repository from SVN like this:

::

    hg convert http://code.sixapart.com/svn/memcached/trunk

This approach takes ages though so you are much better of copying the SVN
repository data to your machine and converting it directly. Here's how I
converted the code for `AuthKit <http://authkit.org>`_:

::

    cd ~
    mkdir svn hg
    scp -r root@authkit.org:/var/svn/AuthKit svn/AuthKit
    hg convert svn/AuthKit hg/AuthKit

The ``hg/AuthKit`` directory is now a mercurial repository. 

Create the file ``hg/AuthKit/.hg/hgrc`` file so it looks something like this:

::

    [web]
    allow_push = james
    style = gitweb
    allow_archive = bz2 gz zip
    description = Python toolkit for web-based authentication, authorisation and permissions (beta).
    contact = James Gardner
    push_ssl = False
 
Now you can make it public by moving it to
``/var/www/vhost/hg.example.com/repo/public`` and setting the correct
permissions:

::

    sudo mv hg/AuthKit /var/www/vhost/hg.example.com/repo/public/
    sudo chown -R www-data:www-data /var/www/vhost/hg.example.com/repo/public/AuthKit

.. tip ::

    If your browser gives you an error about invalid content encodings, it could
    be indicative of a file permission problem. Check the ``repo`` directory and
    everything under it has ``www-data`` owner and group.

You'll need to reload Apache before the new repository is noticed:

::

    sudo /etc/init.d/apache2 reload

See also: http://www.selenic.com/mercurial/wiki/ConvertExtension

Changing the Theme
------------------

The themes are all in
``/var/lib/python-support/python2.5/mercurial/templates/``. To create a new
one, copy an existing one and add a symlink:

::

    sudo mkdir /var/www/vhost/hg.example.com/theme
    sudo cp -pr /var/lib/python-support/python2.5/mercurial/templates/gitweb /var/www/vhost/hg.example.com/theme/custom
    sudo ln -s /var/www/vhost/hg.example.com/theme/custom /var/lib/python-support/python2.5/mercurial/templates/custom 

Now edit the templates in ``/var/www/vhost/hg.example.com/custom``.

Then in the ``.hg/hgrc`` file for the repository you want to theme set:

::

    [web]
    style = custom

You can also change the icon and the stylesheets by changing the files in
``/var/lib/python-support/python2.5/mercurial/templates/static``.

.. warning ::

    The official Mercurial theming documentation suggests you can simply add a
    new variable to your ``map`` file to create a new template which can be used in
    other templates. This doesn't currently work. Instead you can edit
    ``/var/lib/python-support/python2.5/mercurial/templater.py`` and define new
    variables in ``__init__.py`` as keys of the ``self.default`` object. For
    example to create a variable called ``commonbar_top`` you could do this:

    ::
 
        self.defaults['commonbar_top'] = '''
            <p>Top navigation goes here</p>
        '''
 
    You can then use this in any of the templates by adding the string 
    ``#commonbar_top#`` to them.

See also: http://www.selenic.com/mercurial/wiki/Theming

Extra Mercurial Thoughts
------------------------

To avoid the this error:

::

     IOError: sys.stdout access restricted by mod_wsgi

You'll need to make another change. If you just change these lines in
``/var/lib/python-support/python2.5/mercurial/ui.py`` around 386 from this:

::

    def write_err(self, *args):
        try:
            if not sys.stdout.closed: sys.stdout.flush()
            for a in args:
                sys.stderr.write(str(a))
            # stderr may be buffered under win32 when redirected to files,
            # including stdout.
            if not sys.stderr.closed: sys.stderr.flush()
        except IOError, inst:
            if inst.errno != errno.EPIPE:
                raise

So that the Except clause at the end looks like this:

::

    except IOError, inst:
        if not (inst.errno == errno.EPIPE or "access restricted by mod_wsgi" in str(inst)):
            raise

Then everything seems to work nicely.

This is because mod_wsgi restricts access to stdout so you will frequently get errors otherwise.

.. tip ::

   If you get this error:

   ::
 
       Invalid command 'AuthDigestDomain', perhaps misspelled or defined by a module not included in the server configuration

   You have forgotten to enable ``mod_auth_digest`` or you are using the obsolete ``mod_digest``.


You now have a local Mercurial repository. Touch the ``.wsgi`` file to force a reload:

::

    touch /var/www/vhosts/hg.example.com/cgi-bin/hgwebdir.wsgi 



