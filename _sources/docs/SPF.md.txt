# SPF

The Sender Policy Framework is an open standard specifying a technical method to prevent sender address forgery. I plain language, it is a method of informing servers whether a specific mail server is authorised to send an email for a particular domain. With that, it can detect whether an email address is authentic or fake, before allowing it to pass into an inbox. An SPF record increases the likelihood that emails sent by valid addresses are delivered. Without it, genuine email could be categorised as spam. It also protects against malicious emails sent through your domain by spammers.

* A email provider publishes SPF records in the Domain Name System (DNS). These records list which IP addresses are authorised to send email on behalf of their domains.
* In an SPF check, email providers verify the SPF record by looking up the domain name listed in the return path address in the DNS. If the IP address sending email on behalf of the "return path" domain is not listed the record, the message fails authentication.
* SPF records must be kept updated.
* If a message fails SPF check, that doesn’t mean it will always be blocked.
* Breaks when a message is forwarded.
* Does not protect against spoofing of the visible "From" address in messages.

## Add SPF records to DNS

In order that receiving servers can check your SPF record it must be publicly visible. This means publishing it to the DNS server for the chosen domain(s). Go to their domain zone pages and add a new TXT record. For example, to allow mail from all hosts listed in the MX records for the domain:

    v=spf1 mx -all

To allow mail from a specific host:

    v=spf1 a:mail.somedomain.tld -all

Exact format may vary per DNS provider. Check the documentation for the exact style required.

## Installation

The Python SPF policy agent adds SPF policy-checking to Postfix. The SPF record for the sender’s domain for incoming mail will be checked and, if it exists, mail will be handled accordingly.

## Configuration

### policyd-spf.conf 
`/etc/postfix-policyd-spf-python/policyd-spf.conf` looks something like:

    debugLevel = 1
    defaultSeedOnly = 1

    HELO_reject = SPF_Not_Pass
    Mail_From_reject = Fail

    PermError_reject = False
    TempError_Defer = False

    skip_addresses = 127.0.0.0/8,::ffff:127.0.0.0/104,::1

* `debugLevel` controls the amount of information logged by the policy server. The default, level 1, logs no debugging messages, just basic SPF results and errors generated through the policy server.
* The policy server can operate in a test only mode. This allows you to see the potential impact of SPF checking in your mail logs without rejecting mail. Headers are prepended in messages, but message delivery is not affected. This mode is not enabled by default. To enable it, set `TestOnly = 0`. This option was previously named `defaultSeedOnly`. This is still accepted, but logs an error.
* The default HELO check rejection policy is `SPF_Not_Pass`, meaning reject if the SPF result is Fail, Softfail, Neutral, PermError. Not fully RFC 4408 compliant but HELO/EHLO is known first in the SMTP dialogue and there is no reason to waste resources on Mail From checks if the HELO check will already reject the message.

## Postfix integration
### master.cf 

In `/etc/postfix/master.cf` append:

    policyd-spf unix -       n       n       -       0       spawn user=nobody
    argv=/usr/bin/policyd-spf
    
### main.cf

To increase the Postfix policy agent timeout, which will prevent Postfix from aborting the agent if transactions run a bit slowly

    policyd-spf_time_limit = 3600

And append `check_policy_service unix:private/policyd-spf` **after** `reject_unauth_destination`, for example:

    smtpd_recipient_restrictions = reject_non_fqdn_recipient,reject_unknown_recipient_domain,permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination,reject_non_fqdn_sender,reject_unlisted_recipient,check_policy_service unix:private/policyd-spf

### Restart postfix 

    # systemctl restart postfix

## Testing

* Check the operation of the policy agent by looking at raw headers on incoming email messages for the SPF results header.
* The SPF policy agent also logs to `/var/log/mail.log`. In the mail.log file you’ll see messages like this from the policy agent

## Configuration resources

  * [Man policyd-spf](https://manpages.debian.org/testing/postfix-policyd-spf-python/policyd-spf.conf.5.en.html) - policyd-spf python configuration parameters 
  * [Kitterman Technical Services](https://www.kitterman.com/spf/validate.html) offers tools for setting up an SPF record.
  * [appmaildev.com](https://appmaildev.com/en/spf/) also allows for testing SPF records.
  * [MX Toolbox](https://mxtoolbox.com/) allows users to [look up their SPF records](https://mxtoolbox.com/spf.aspx), making it possible to publish a list of domains that are authorised to send an email on their behalf, and a [check if your mail server is blacklisted](https://mxtoolbox.com/blacklists.aspx).
