.. Mailserver mitigations documentation master file, created by
   sphinx-quickstart on Sun Jun 26 04:57:04 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Mailserver mitigations
==================================================

* Use multiple listeners for each interface and correlate them with certain allow and deny rules.
* When having an external facing mailserver and an internal mailserver, relay rules based on sender address/recipient address, or relaying for authenticated users only can be implemented dependend on context and purpose (but NEITHER is to accept open relaying).
* Postfix is a MTA (Mail Transfer Agent) that sends and receives emails to and from other computers on the network using the Simple Mail Transfer Protocol (SMTP). It is widely used, well documented, and actively maintained and developed. It requires minimal configuration and is efficient with system resources.
* Having installed the Postfix mail server to operate as the Simple Mail Transfer Protocol (SMTP) service, to retrieve the incoming mail from the server, POP/IMAP are used by a client to read messages from an email server. Dovecot and Cyrus are the two prime candidates.
* Defend the mailserver against spam and abuse, and get the proper credentials so that other e-mail providers do not perceive the server as a spammer.
* By implementing several authentication methods, a mail server can have better communication security and be better equipped against attacks and unauthorised access.
* IMAP servers like Dovecot and Cyrus support server-side mail filtering through the implementation of a mail filtering language called Sieve.
* SASL (Simple Authentication and Security Layer) is a framework that can be used by many connection-oriented Internet protocols for securing data, servers and users. With SASL enabled, Postfix will not accept any incoming SMTP connections without proper authentication.
* Host control ensures only valid emails are further processed by a mail server. A well known method is SPF (Sender Policy Framework). A more complex authentication method is DKIM (Domain Keys Identified Mail Signature). The email header contains an encrypted signature and is in its turn encrypted, pointing to an encrypted key, published on DNS servers by the sending domain. The server processing the email will use this key to decode the email body. If the decryption is successful, then the email is valid.
* The new kid on the block is DANE (DNS-based Authentication of Named Entities). If you have a DNS host that supports DNS-based Authentication of Named Entities (DANE) TLSA records, consider upgrading your DNSSEC. Postfix supports DANE.
* Instead of using Sieve, Spamassassin, (or Amavis, Pyzor and Razor) and OpenDKIM, Rspamd can be used.
* Limiting the number of connection and authentication errors, the maximum number of commands or setting a time-out for your sessions can help protect a mail server from further DOS attacks.
* After a user is authenticated over Simple Message Transfer Protocol (SMTP), there will be no automatically encrypted connection. Commands and emails are exchanged with the server in plain text, allowing a man-in-the-middle attacker to read and modify the communication and inject new commands. Which is why we want TLS (Transport Level Security).

.. toctree::
   :glob:
   :maxdepth: 1
   :includehidden:
   :caption: Common mitigations

   docs/*

.. toctree::
   :caption: All mitigations

   Overview <https://tymyrddin.github.io/mitigations/>
