# Cyrus

The Cyrus IMAP server is electronic mail server open source software developed by Carnegie Mellon University. It differs from other Internet Message Access Protocol (IMAP) server implementations in that it is generally intended to be run on sealed servers, where normal users cannot log in. And it is not lightweight.

## Installation

    # apt-get install cyrus-imapd cyrus-sasl cyrus-sasl-plain

You will be asked to set a password for the default administrative user `cyrus`

## Get SSL certificate

Using Let’s Encrypt, request a certificate for the mail server by (replacing mail.mydomain.com with the fully qualified domain name (FQDN) of your server):

    certbot certonly --standalone -d mail.mydomain.com

## Integration with postfix

## Firewall

To access the mail server from another computer, configure the server firewall to allow connections to the server on the necessary ports. The default ports are IMAP - 143; IMAPS - 993; POP3 - 110; and POP3S - 995.

## Configuration resources

  * [Cyrus IMAP](https://www.cyrusimap.org/index.html)

