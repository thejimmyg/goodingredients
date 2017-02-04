Locales
+++++++

It is important you set up locales for all the applications which may need them.

This avoids problems such as this which you might get otherwise:

::

    perl: warning: Setting locale failed.
    perl: warning: Please check that your locale settings:
    	LANGUAGE = (unset),
    	LC_ALL = (unset),
    	LANG = "en_GB.UTF-8"
        are supported and installed on your system.
    perl: warning: Falling back to the standard locale ("C").

Install ``locales``:

::

    $ sudo apt-get install locales

You then configure locales with:

::

    $ sudo dpkg-reconfigure locales

Here's what you get:

::

    Package configuration
    
     ┌──────────────────────────┤ Configuring locales ├──────────────────────────┐
     │ Locales are a framework to switch between multiple languages and allow    │ 
     │ users to use their language, country, characters, collation order, etc.   │ 
     │                                                                           │ 
     │ Please choose which locales to generate. UTF-8 locales should be chosen   │ 
     │ by default, particularly for new installations. Other character sets may  │ 
     │ be useful for backwards compatibility with older systems and software.    │ 
     │                                                                           │ 
     │ Locales to be generated:                                                  │ 
     │                                                                           │ 
     │    [ ] en_DK.ISO-8859-15 ISO-8859-15                                      │ 
     │    [ ] en_DK.UTF-8 UTF-8                                                  │ 
     │    [ ] en_GB ISO-8859-1                                               ▒   │ 
     │    [ ] en_GB.ISO-8859-15 ISO-8859-15                                  ▒   │ 
     │    [*] en_GB.UTF-8 UTF-8                                                  │ 
     │                                                                           │ 
     │                                                                           │ 
     │                    <Ok>                        <Cancel>                   │ 
     │                                                                           │ 
     └───────────────────────────────────────────────────────────────────────────┘ 
                                                                                   
I choose to make the new locale the default by highlighting ``en_GB.UTF-8`` and chosing ``Ok``:

::

    Package configuration
    
    
     ┌──────────────────────────┤ Configuring locales ├──────────────────────────┐
     │ Many packages in Debian use locales to display text in the correct        │ 
     │ language for the user. You can choose a default locale for the system     │ 
     │ from the generated locales.                                               │ 
     │                                                                           │ 
     │ This will select the default language for the entire system. If this      │ 
     │ system is a multi-user system where not all users are able to speak the   │ 
     │ default language, they will experience difficulties.                      │ 
     │                                                                           │ 
     │ Default locale for the system environment:                                │ 
     │                                                                           │ 
     │                                None                                       │ 
     │                                en_GB.UTF-8                                │ 
     │                                                                           │ 
     │                                                                           │ 
     │                    <Ok>                        <Cancel>                   │ 
     │                                                                           │ 
     └───────────────────────────────────────────────────────────────────────────┘ 
                                                                                   

You'll see this output and it may take a little while to generate the locales:

::

    Generating locales (this might take a while)...
      en_GB.UTF-8... done
    Generation complete.


