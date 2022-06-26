# SASL

Once Postfix is up and running, add SASL authentication to avoid open relaying. In order to prevent anonymous users from spamming, only authenticated and trusted users will be able to send emails. Postfix supports two SASL implementations: Cyrus SASL (SMTP client and server) and Dovecot SASL (SMTP server only). Both implementations can be built into Postfix simultaneously.

## Cyrus

Install:

    # apt-get install libsasl2-modules sasl2-bin

SASL can use different authentication methods. The default one is PAM (as configured in `/etc/conf.d/saslauthd`), but to set it up properly you have to create `/etc/sasl2/smtpd.conf`. Pambase 20190105.1-1 and newer use restrictive fallback for "other" PAM service, and a pam configuration file is required.

Create a file `/etc/postfix/sasl/smtpd.conf`:

    pwcheck_method: saslauthd
    mech_list: PLAIN LOGIN
    log_level: 7

Setup a separate saslauthd process to be used from Postfix. Create a copy of saslauthd's config file:

    # cp /etc/default/saslauthd /etc/default/saslauthd-postfix

Append:

    START=yes
    DESC="SASL Auth. Daemon for Postfix"
    NAME="saslauthd-postf"      # max. 15 char.
    # Option -m sets working dir for saslauthd (contains socket)
    OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd"        # postfix/smtp in chroot()

Create the required subdirectories in postfix chroot directory:

    dpkg-statoverride --add root sasl 710 /var/spool/postfix/var/run/saslauthd

Add user `postfix` to the group `sasl`:

    adduser postfix sasl

Restart saslauthd:

    # systemctl saslauthd  restart
    [ ok ] Stopping SASL Auth. Daemon: saslauthd.
    [ ok ] Stopping SASL Auth. Daemon for Postfix: saslauthd-postf.
    [ ok ] Starting SASL Auth. Daemon: saslauthd.
    [ ok ] Starting SASL Auth. Daemon for Postfix: saslauthd-postf.

Edit `/etc/postfix/main.cf`: 

    smtpd_sasl_local_domain = $myhostname
    smtpd_sasl_auth_enable = yes
    broken_sasl_auth_clients = yes
    smtpd_sasl_security_options = noanonymous
    smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination

Restart (reloading is not enough):

    # systemctl postfix restart

### SMTP

To enable SASL for accepting mail from other users, in `/etc/postfix/master.cf` under the submission or smtp section (depending on what is used) append/uncomment:

    submission inet n       -       n       -       -       smtpd
      -o syslog_name=postfix/submission
      -o smtpd_tls_security_level=encrypt
      -o smtpd_sasl_auth_enable=yes
      -o smtpd_reject_unlisted_recipient=no
    #  -o smtpd_client_restrictions=$mua_client_restrictions
    #  -o smtpd_helo_restrictions=$mua_helo_restrictions
    #  -o smtpd_sender_restrictions=$mua_sender_restrictions
      -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
      -o milter_macro_daemon_name=ORIGINATING

* The three restriction options (client, helo, sender) can also be left commented out, since `smtpd_recipient_restrictions` already handles SASL users.

Start/enable the saslauthd service and restart the postfix service.

## Dovecot

If Dovecot is used as IMAP or POP mail server and **all users** already authenticate (such as with [PAM](https://tymyrddin.github.io/linux-server-mitigations/docs/pki/PAM.html)), then there is no need to configure another package.

### SMTP

Edit `/etc/postfix/master.cf` and append/uncomment under the submission or smtp section (depending on what is used):

    # SASL authentication with dovecot
      -o smtpd_tls_security_level=encrypt
      -o smtpd_sasl_auth_enable=yes
      -o smtpd_sasl_type=dovecot
      -o smtpd_sasl_path=private/auth
      -o smtpd_sasl_security_options=noanonymous
      -o smtpd_sasl_local_domain=$myhostname
      -o smtpd_client_restrictions=permit_sasl_authenticated,reject
      -o smtpd_recipient_restrictions=reject_non_fqdn_recipient,reject_unknown_recipient_domain,permit_sasl_authenticated,reject

In `/etc/postfix/main.cf`:

    smtpd_relay_restrictions = permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination
    smtpd_sasl_local_domain = $myhostname

* `smtpd_sender_restrictions` filters mails based on the MAIL FROM command; This command is easy faked by telneting an open relay and typing in this command, therefore mail counl be sent with a valid MAIL FROM address. Use `smtpd_client_restrictions` instead, which checks the hostname or IP address of the smtpd client (the other MTA/SMTP connecting to the internal smtpd) in a black list, if listed mail is denied.
* If mails are marked as `NoBounceOpenRelay` try:

    smtpd_sasl_authenticated_header = yes

To configure Postfix to enable email clients to connect to the SMTP server. In `/etc/postfix/main.cf`:

    smtpd_sasl_auth_enable = yes
    smtpd_sasl_security_options = noanonymous
    smtpd_sasl_local_domain = $myhostname
    smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks, reject_unauth_destination
    broken_sasl_auth_clients = yes
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth

* `broken_sasl_auth_clients` enables interoperability with remote SMTP clients that implement an obsolete version of the AUTH command. Default is `no`. Setting it to `yes` makes Postfix advertise AUTH support in a non-standard way.

Restart both dovecot and postfix.

## Configuration resources

* [Postfix and Dovecot SASL](https://wiki2.dovecot.org/HowTo/PostfixAndDovecotSASL)

