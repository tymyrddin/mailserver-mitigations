# TLS

* Email servers wrap SMTP via direct TLS or a connection upgrade with STARTTLS at ports 465/587
* A Postfix SMTP server needs a certificate and a private key in `.pem` format. The private key must not be encrypted (= must be accessible without a password). 
* Public Internet MX hosts without certificates signed by a well-known public CA must still generate, and be prepared to present to most clients, a self-signed or private-CA signed certificate. 
* For non-public Internet MX hosts, Postfix supports configurations with no certificates. This entails the use of anonymous TLS ciphers, which are not supported by typical SMTP clients.

## Creating keys and certificates
By default, Postfix will not accept secure mail. To set Postfix up to accept secure mail, obtain a certificate. A certificate can be obtained either from a certificate authority with a Certificate Signing Request (CSR) or Let's Encrypt certbot, or self-signed. Self-signed certificates can be generated easily, but clients will reject them by default, unless each and every client is configured to trust the self-signed certificate. 

Point Postfix to your TLS certificates by appending to `/etc/postfix/main.cf`: 

    smtpd_tls_cert_file = /etc/letsencrypt/live/$domain.$tld/fullchain.pem
    smtpd_tls_key_file = /etc/letsencrypt/live/$domain.$tld/privkey.pem
    smtpd_tls_dh1024_param_file = /etc/ssl/private/2048.dh
    smtpd_tls_dh512_param_file = /etc/ssl/private/512.dh

If you want the Postfix SMTP server to accept remote SMTP client certificates issued by one or more root CAs, append the root certificate to `$smtpd_tls_CAfile` or install it in the `$smtpd_tls_CApath` directory. 

    smtpd_tls_CApath = /etc/ssl/certs

## Configuration

### SMTP (sending)

By default, Postfix/sendmail will not send email encrypted to other SMTP servers. To use TLS when available, in `/etc/postfix/main.cf`, set:

    smtp_tls_security_level = may

Sending AUTH data over an unencrypted channel poses a security risk. When TLS layer encryption is not required but optional (`smtpd_tls_security_level = may`), it is still useful to only offer AUTH when TLS is active. To maintain compatibility with non-TLS clients, the default is to accept AUTH without encryption. In order to change this behaviour: 

In `/etc/postfix/main.cf`:
    smtpd_tls_auth_only = yes

### SMTP (receiving)

There are two ways to accept secure mail. 

For STARTTLS over SMTP (port 587) uncomment in `/etc/postfix/master.cf`:

    submission inet n       -       n       -       -       smtpd
      -o syslog_name=postfix/submission
      -o smtpd_tls_security_level=encrypt
      -o smtpd_sasl_auth_enable=yes
      -o smtpd_tls_auth_only=yes
      -o smtpd_reject_unlisted_recipient=no
    #  -o smtpd_client_restrictions=$mua_client_restrictions
    #  -o smtpd_helo_restrictions=$mua_helo_restrictions
    #  -o smtpd_sender_restrictions=$mua_sender_restrictions
      -o smtpd_recipient_restrictions=
      -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
      -o milter_macro_daemon_name=ORIGINATING

For SMTPS (port 465), uncomment in `/etc/postfix/master.cf`:

    smtps     inet  n       -       n       -       -       smtpd
      -o syslog_name=postfix/smtps
      -o smtpd_tls_wrappermode=yes
      -o smtpd_sasl_auth_enable=yes
      -o smtpd_reject_unlisted_recipient=no
    #  -o smtpd_client_restrictions=$mua_client_restrictions
    #  -o smtpd_helo_restrictions=$mua_helo_restrictions
    #  -o smtpd_sender_restrictions=$mua_sender_restrictions
      -o smtpd_recipient_restrictions=
      -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
      -o milter_macro_daemon_name=ORIGINATING


When the TLS information is not included in the headers of the message it has processed from another server or SMTP client that uses TLS, it can be added with:

    smtpd_tls_received_header = yes

The "smtp server" part of Postfix (the first daemon that listens on port 25) then adds the TLS information to the "Received" message header of an incoming message.

### Logging

To get more information about Postfix SMTP client TLS activity you can increase the loglevel from 0..4. Each logging level also includes the information that is logged at a lower logging level.

    0 Disable logging of TLS activity.
    1 Log TLS handshake and certificate information.
    2 Log levels during TLS negotiation.
    3 Log hexadecimal and ASCII dump of TLS negotiation process.
    4 Log hexadecimal and ASCII dump of complete transmission after STARTTLS.

In `/etc/postfix/main.cf`:
    smtpd_tls_loglevel = 0

### Cipher controls

The Postfix SMTP server supports 5 distinct cipher grades as specified by the `smtpd_tls_mandatory_ciphers` parameter, which determines the minimum cipher grade with mandatory TLS encryption. Default is medium.

    smtpd_tls_mandatory_ciphers = high

Cipher grade `high` enables only "HIGH" grade OpenSSL ciphers specified in the tls_high_cipherlist configuration parameter, which you are strongly encouraged to not change. 

The `smtpd_tls_ciphers` parameter controls the minimum cipher grade used with opportunistic TLS. Default is `medium`.

    smtpd_tls_ciphers = medium

By default, anonymous ciphers are enabled. They are automatically disabled when remote SMTP client certificates are requested. For excluding cipher types from the SMTP server cipher list at all TLS security levels:

    smtpd_tls_exclude_ciphers = aNULL, MD5, DES, 3DES, DES-CBC3-SHA, RC4-SHA, AES256-SHA, AES128-SHA, eNULL, EXPORT, RC4, PSK, aECDH, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CDC3-SHA, KRB5-DE5, CBC3-SHA

The Postfix SMTP server security grade for ephemeral elliptic-curve Diffie-Hellman (EECDH) key exchange: 

    smtpd_tls_eecdh_grade = strong

`strong` means use EECDH with approximately 128 bits of security at a reasonable computational cost. This is the current best-practice trade-off between security and computational efficiency. 

By default, the OpenSSL server selects the client's most preferred cipher-suite that the server supports. With SSLv3 and later, the server may choose its own most preferred cipher-suite that is supported (offered) by the client. The default OpenSSL behaviour applies with "tls_preempt_cipherlist = no" To enable server cipher-suite preferences:

    tls_preempt_cipherlist = yes

### Entropy

The security of cryptographic software such as TLS depends critically on the ability to generate unpredictable numbers for keys and other information. The `tlsmgr` process maintains a Pseudo Random Number Generator (PRNG) pool for the purpose. This is queried by the `smtp` and `smtpd` processes when they initialize. By default, these daemons request 32 bytes, the equivalent to 256 bits, more than enough to generate a 128 bit (or 168 bit) session key. A good entropy source is `/dev/urandom`

    tls_random_source = dev:/dev/urandom

### Session cache

The remote SMTP server and the Postfix SMTP client negotiate a session, which takes some computer time and network bandwidth.
Cached Postfix SMTP server session information expires after a certain amount of time. Postfix/TLS does not use the OpenSSL default of 300s, but a longer time of 3600sec. RFC 2246 even recommends a maximum of 24 hours. 

    smtpd_tls_session_cache_timeout = 3600s

### Obsolete TLS parameters 

    smtpd_tls_mandatory_protocols = TLSv1

## Firewall

Allow connections to port 587 and 465 (depending) by opening the port for your server in the server firewall.

## Configuration resources

  * [Guide to Deploying Diffie-Hellman for TLS](https://weakdh.org/sysadmin.html)


