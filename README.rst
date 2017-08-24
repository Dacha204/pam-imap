pam-imap for 2017+
==========================
This is an updated version of pam_imap 0.3.8, taken from
http://pam-imap.sourceforge.net/. See README.original for the original
readme.

These files were mostly updated last in 2003, 2004 and imap.c in 2009.
These edits might make it work in 2017 again.

**This fork fix compile and linking issues on Ubuntu 16.04.**


Changes
--------
* Compile & linking fixes
* New example configs for Ubuntu (original were for CentOS/RHEL)
* Cleaner config file
* Changed default config file location to /etc/pam_imap.conf


See also: 

* https://github.com/MrDroid/pam_imap
* https://github.com/wdoekes/pam-imap

Compiling
---------

Debian install prerequisites: libpam0g-dev libssl-dev libgdbm-dev dh-autoreconf

.. code-block:: console

    $ sudo apt-get install libpam0g-dev libssl-dev libgdbm-dev dh-autoreconf

.. code-block:: console

    $ ./bootstrap
    $ ./configure CPPFLAGS=-DVERIFY_CERT
    $ make

> **Note:**
> To disable server certificate check, ommit **-DVERIFY_CERT** flag

Compiling with debug info

.. code-block:: console

    $ ./bootstrap
    $ ./configure CPPFLAGS=-DVERIFY_CERT --with-debug=yes
    $ make

Building a Debian package with ``gbp``:

.. code-block:: console

    $ git clean -xf
    $ gbp buildpackage -us -uc -sa --git-debian-branch=master --git-upstream-tag='v%(version)s' --git-ignore-new

Install & Setup
------------------
* Copy files

.. code-block:: console

    $ sudo cp pam_imap.so /lib/*/security  (/lib/x86_64-linux-gnu/security)
    $ sudo cp conf/pam_imap.conf /etc/

* Download **cacert.pem** from https://curl.haxx.se/docs/caextract.html (The Mozilla CA certificate store in PEM format) or your IMAP server certificate in /etc/ssl/certs/

.. code-block:: console

    $ sudo wget -O /etc/ssl/certs/cacert.pem https://curl.haxx.se/ca/cacert.pem

* Modify ``/etc/pam_imap.conf``

.. code-block:: text
    
    Example pam_imap.conf:

        PAM_PasswordString = Password:
        CertificateFile /etc/ssl/certs/cacert.pam
        PAM_Server0 = imaps:imap.gmail.com:993
        PAM_Domain = gmail.com
        PAM_BlockList = root, admin, Administrator, apache
        PAM_HashEnable = no
        PAM_HashFile = /etc/pam_imap.gdbm
        PAM_HashDelta = 20

* Modify PAM configuration in ``/etc/pam.d`` for your needs.

For testing you can use ``check_user`` utility and *check_user* config:

.. code-block:: console

    $ sudo cp conf/check_user /etc/pam.d/
    ... modify config /etc/pam.d/check_user ...
    $ sudo ./check_user <username_to_test>
    
Usage: ``check_user <username> [service]``, where service is service is the pam service in /etc/pam.d/<service>. Default service is *check_user*

Example configs
--------------

It is recommended that you first use ``check_user`` utility and config to test your PAM config stack before you apply on desired service. You don't want to accidentally lock yourself out.

Goals for this examples:

* User can use his Gmail credentials to login on workstation
* Only certain users can login to workstation
* (optional) User can login to workstation if Internet is down / imap server is unreachable

How to implement that?

Lets say **john.doe** @gmail.com is our user username. 

On workstation we will make new user **john.doe**  (or rename current one to **john.doe**). After we authenticate user against IMAP server, we check if the user has local account on workstation. That way we are preventing any user with valid email/pass to login to workstation. Only users who have local account with same username can use workstation.

> **Note:**
> By default Ubuntu forbids "." in username. Use ``--force-badname`` when creating user with ``adduser``

**pam_imap.conf** content:

.. code-block:: text

    PAM_PasswordString = Password:
    CertificateFile /etc/ssl/certs/cacert.pam
    PAM_Server0 = imaps:imap.gmail.com:993
    PAM_Domain = gmail.com
    PAM_BlockList = root, admin, Administrator, apache
    PAM_HashEnable = no
    PAM_HashFile = /etc/pam_imap.gdbm
    PAM_HashDelta = 20

For PAM service config we want minimal changes how default config works. On Ubuntu all services (login,sshd,su..) include *common-auth, common-account, common-password, common-session"*. We want our changes to be visible everywhere, so we are going to change some *common-* file(s).

We are changing how user authenticate and how is authorized to use workstation, so only *common-auth* and *common-account* are changed.

For start let's make *check_user* config to test our modification first, like this: 

/etc/pam.d/check_user content:

.. code-block:: text

    ## From common-auth
    #auth    [success=1 default=ignore]      pam_unix.so nullok_secure
    auth    sufficient      pam_unix.so     nullok try_first_pass    
    auth    sufficient      pam_imap.so     conf=/etc/pam_imap.conf
    auth    requisite       pam_deny.so
    auth    required        pam_permit.so

    ## From common-account
    #account [success=1 new_authtok_reqd=done default=ignore]        pam_unix.so  
    account required        pam_unix.so
    account sufficient                      pam_localuser.so
    account requisite                       pam_deny.so
    account required                        pam_permit.so

    @include common-password
    @include common-session


First we try to authenticate with local user username/password. If that fails, then we try to authenticate via IMAP. PAM_IMAP concats '@gmail' on username and send that as username to imap server. If that succeeded, user is successfully authenticated. 

Then user is checked if it is authorized to use workstation. pam_localuser checks if user has local account on workstation, and pam_unix checks if local account is enabled and other constraints.  

Only if both succeeded, user is able to login to workstation.

Lets check our config:

.. code-block:: console

    sudo ./check_user john.doe

Output (if you compiled with debug output)

.. code-block:: text

    user=john.doe
    Password: 
    config_file=/etc/pam_imap.conf
    Reading configuration file /etc/pam_imap.conf
    ********************************
    Debug-Option printout: 
    port=993
    host=imap.gmail.com
    box=INBOX
    require_ssl=1
    use_imaps=1
    use_sslv2=1
    use_sslv3=1
    use_tlsv1=1
    cert_file=/etc/ssl/certs/cacert.pem
    user=john.doe@gmail.com
    ********************************
    Resolving imap.gmail.com... ok
    Connecting to 74.125.206.109:993... ok
    SSL support enabled
    Logging in...
    server_connect() returned: 0 PAM_SUCCESS
    pam_sm_authenticate: returning PAM_SUCCESS
    check_user: pam_authenticate() returned: 0: PAM_SUCCESS

    check_user end result:
    ########################
    Authenticated
    Account Authorized
    ########################

If everything works after testing with ``check_user`` then we can modify ``common-auth`` and ``common-account``

.. code-block:: text

    #auth   [success=1 default=ignore]      pam_unix.so nullok_secure
    auth    sufficient                      pam_unix.so nullok
    auth    sufficient                      pam_imap.so conf=/etc/pam_imap.conf
    auth    requisite                       pam_deny.so
    auth    required                        pam_permit.so

.. code-block:: text

    #account        [success=1 new_authtok_reqd=done default=ignore]        pam_unix.so 
    account required        pam_unix.so
    account sufficient      pam_localuser.so
    account requisite                       pam_deny.so
    account required                        pam_permit.so

Now we can login to workstation using gmail password. Works also for SSH, sudo, su...