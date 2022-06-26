# Dovecot

Dovecot is an open source IMAP and POP3 server for Linux/UNIX-like systems, written primarily with security in mind. Dovecot primarily aims to be a lightweight, fast and easy to set up open source mailserver. The below is for debian 9 (probably also works on buster)

## Installation 

    apt-get install dovecot-common dovecot-imapd dovecot-pop3d

## Get SSL certificate

Using Let’s Encrypt, request a certificate for the mail server by (replacing mail.mydomain.com with the fully qualified domain name (FQDN) of your server):

    certbot certonly --standalone -d mail.mydomain.com

## Configuration

Dovecot's configuration files are in `/etc/dovecot/conf.d/`. The default configuration is ok for most systems, but make sure to read through the configuration files to see what options are available. 

By default dovecot will try to detect what mail storage system is in use on the system. To use the Maildir format edit `/etc/dovecot/conf.d/10-mail.conf` to set it:
    
    mail_location = maildir:~/Maildir

To configure it for the SSL certificates, open `/etc/dovecot/conf.d/10-ssl.conf` and append:

    ssl = required
    ssl_cert = </etc/letsencrypt/live/mail.mydomain.com/fullchain.pem
    ssl_key = </etc/letsencrypt/live/mail.mydomain.com/privkey.pem

To force SSL/TLS encryption, open `/etc/dovecot/conf.d/10-auth.conf` and make sure:

    disable_plaintext_auth = yes

### SASL
In the main configuration file `/etc/dovecot/conf.d/10-master.conf`, uncomment the paragraph for Postfix in the auth service block:

    service auth {
      # Postfix smtp-auth
      unix_listener /var/spool/postfix/private/auth {
        mode = 0660
      }
    }

This will create the `private/auth` path for the SASL configuration of Postfix. Because Postfix runs chrooted in `/var/spool/postfix`, a relative path must be used.

## Integration with postfix

In `/etc/postfix/master.cf` append:

    dovecot   unix  -       n       n       -       -       pipe
  flags=DRhu user=email:email argv=/usr/lib/dovecot/deliver -f ${sender} -d ${recipient}
  
## Firewall

To access the mail server from another computer, configure the server firewall to allow connections to the server on the necessary ports. The default ports are IMAP - 143; IMAPS - 993; POP3 - 110; and POP3S - 995.

## Configuration resources

* [Dovecot configuration](https://wiki2.dovecot.org/#Dovecot_configuration)
* [Dovecot Quick Configuration](https://wiki2.dovecot.org/QuickConfiguration)

