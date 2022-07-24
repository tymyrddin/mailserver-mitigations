# DANE

DNS-based Authentication of Named Entities (DANE) is an Internet security protocol to allow X.509 digital certificates, commonly used for Transport Layer Security (TLS), to be bound to domain names using Domain Name System Security Extensions (DNSSEC). TLSA resource record is an own type of DNS record.

* [Postfix](Postfix.md) supports DANE
* In Postfix, certificate usage 0 is unsupported, 1 is mapped to 3, and 2 is optional, thus it is recommended to publish a "3" record. 

## Configuration

In `/etc/postfix/main.cf`

  smtpd_use_tls = yes
  smtp_dns_support_level = dnssec
  smtp_tls_security_level = dane

In `/etc/postfix/master.cf`

    dane       unix  -       -       n       -       -       smtp
      -o smtp_dns_support_level=dnssec
      -o smtp_tls_security_level=dane

### Multiple domains

To use per-domain policies, for example for opportunistic DANE for domain.org and mandatory DANE for domain.com:

In `/etc/postfix/main.cf`

    indexed = ${default_database_type}:${config_directory}/

    # Per-destination TLS policy
    #
    smtp_tls_policy_maps = ${indexed}tls_policy

    # default_transport = smtp, but some destinations are special:
    #
    transport_maps = ${indexed}transport

In `transport`:

    domain.com dane
    domain.org dane

In `tls_policy`:

    domain.com dane-only

## Configuration resources

  * [DANE: Common mistakes](https://dane.sys4.de/common_mistakes)
  * [DANE validator](https://danetools.com/dane)
  * [Generate TLSA Record online](https://www.huque.com/bin/gen_tlsa)
  * [RFC6698: The DNS-Based Authentication of Named Entities (DANE) Transport Layer Security (TLS) Protocol: TLSA](https://tools.ietf.org/html/rfc6698)
  * [SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE) Transport Layer Security (TLS)](https://tools.ietf.org/html/rfc7672)


