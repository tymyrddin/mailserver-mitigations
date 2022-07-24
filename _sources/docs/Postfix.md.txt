# Postfix

Postfix is a mail transfer agent (MTA). The below was for debian 9 (will most likely also work with later versions).

* The two most important configuration files are:
    * `/etc/postfix/master.cf`, defining what Postfix services are enabled and how clients connect to them
    * `/etc/postfix/main.cf`, the main configuration file, the bulk, the meat.
* Most configuration changes only need a reload of postfix in order to take effect. Only some require a restart.
* Don't manage mailboxes from postfix.
* Check configuration with the `postfix check` command.
* To see all of the configuration files use the `postconf` command.
* To see how the setup differs from the defaults, use `postconf -n`.
* On debian, Postfix runs by default in a chroot jail. On other distro's where this is not the case, it is recommended to set it up in a jail.
* Limit use of Postfix services
* Restrict which hosts are granted or refused access with the `smtpd_*_restrictions` directives.
* The `vrfy` command allows an attacker to determine if an account exists on a system, providing significant assistance to a brute force attack on user accounts. `vrfy` may provide additional information about users on the system, such as the full names of account owners. Default is `no`. Set to `disable_vrfy_command = yes`.
* Client control rules can be based on the sender or receiver address and limitations of the number and size of email messages.
* Consider greylisting: In greylisting the server sends a request to have the email resent, after temporarily rejecting the email. The server saves in a list the sender IP and the recipient and returns a temporary error. All valid servers will then resend the emails, spamming scripts will not.
* Other efficient protocol level solutions in Postfix can be used to make sure the mail server is RFC compliant and prevent email looping (a very simple method would be setting a maximum numbers of "Received" headers per email).
* With policy services Postfix' behaviour of mail delivery can be fine-tuned. Making changes to configurations of such services require a restart of Postfix.

## Installation 

    # apt-get install postfix

You will be asked a series of questions. On the first prompt, select //Internet Site// option as the general type for Postfix configuration, continue, and then add domain name to system mail name.
 
## Basic configuration 

Configure the basics:

    # cp /etc/postfix/main.cf{,.old}
    # vi /etc/postfix/main.cf


    myhostname = $hostname                                   //Use command "hostname" to display your hostname
    myorigin = $mydomain                                     //Add domain, so others can not abuse the mailsystem
    relay_domains = domain1.com, domain2.com, domain3.com    //Add the domains the system will handle

### Multiple local domains 

Postfix can be configured for more than one domain via the use of a hash file. This file contains the list of domains postfix will accept for local delivery. Open the `/etc/postfix/main.cf` configuration file and append:

    virtual_alias_domains = hash:/etc/postfix/virtual_domains

Create the file, and add all the domains postfix should accept in it (one per line). The postmap program expects the file to have 2 columns but the second column is ignored. Do add that second column and add comments like `#something` in it. It will work without, but you get warnings.

To make the hashfile:

    # postmap /etc/postfix/virtual_domains

### Remote access 
The postfix SMTP server access table contains the list for access control for remote SMTP clients: 

Open the `/etc/postfix/main.cf` configuration file and append:

    check_sender_access = hash:/etc/postfix/access

Create the file, and add all the host names, network addresses, and envelope sender or recipient addresses postfix should accept in it (one per line)

To make the hashfile:

    # postmap /etc/postfix/access

## Managing mailboxes 

**Don't manage mailboxes from postfix.** Redirect messages for delivery via POP/IMAP server. In case of dovecot there is `dovecot-lda` aka `deliver` that do everything and much more, like user-controlled message filtering, quota management, autoreplying etc. Most modern POP/IMAP servers have a lot of utilities for common tasks in infrastructures.

Maildir **is** the newer and preferrable format due to the lot of improvements comparatively to mailbox. It has an index for each folder that allow to control duplicates, expiration times and even full-text search and is faster on a huge pile of messages. Dovecot can easily operate maildir with 300k messages in it without any visible slowdown.

## Abuse and spam 

### SMTP restrictions 
The `smtpd_*_restrictions` directives can be used to set what data is accepted for any SMTP command, for example in `/etc/postfix/main.cf`: 

    # smtpd_recipient_restrictions = reject_invalid_hostname,
        reject_unknown_recipient_domain,
        reject_unauth_destination,
        reject_rbl_client sbl.spamhaus.org,
        permit

    # smtpd_helo_restrictions = reject_invalid_helo_hostname,
        reject_non_fqdn_helo_hostname,
        reject_unknown_helo_hostname
        
### Blacklisting 

Manually blacklisting incoming emails by sender address can easily be done with Postfix. Create and open an `/etc/postfix/blacklist_incoming` file and append sender email addresses:

user@domain.com REJECT

To create a database, use postmap command:

    # postmap hash:blacklist_incoming

And append **before the first permit rule** in `/etc/postfix/main.cf`:

    smtpd_recipient_restrictions = check_sender_access hash:/etc/postfix/blacklist_incoming

Use of others' lists is also possible. An RBL list is a spammer blacklist of domains. In `/etc/postfix/main.cf`:

    smtpd_client_restrictions = reject_rbl_client dnsbl.sorbs.net

### Greylisting 

Install the postgrey package. Open up `/etc/postfix/main.cf` and append

    smtpd_recipient_restrictions = check_policy_service inet:127.0.0.1:10030

Then start/enable the `postgrey` service and after that reload the postfix service. 

Its configuration is done via editing the postgrey.service file. Copy it over to edit it.

    # cp /usr/lib/systemd/system/postgrey.service /etc/systemd/system/

Using postgrey it is also possible to add automatic whitelisting based on successful deliveries. These then don't have to wait any more. This can be done by , you could add the adding the `--auto-whitelist-clients=5` (default is 5) but the preferred method is the override:

    # cat /etc/systemd/system/postgrey.service.d/override.conf
    [Service]
    ExecStart=
    ExecStart=/usr/bin/postgrey --inet=127.0.0.1:10030 \
           --pidfile=/run/postgrey/postgrey.pid \
           --group=postgrey --user=postgrey \
           --daemonize \
           --greylist-text="Greylisted for %%s seconds" \
           --auto-whitelist-clients

Add your own list of whitelisted clients in addition to the default ones by creating the file `/etc/postfix/whitelist_clients.local` (one host or domain per line).

Restart the postgrey.service for the changes to take effect. 

### Header filtering 

Postfix header or body_checks are designed to stop a flood of mail from worms or viruses. Optional lookup tables for content inspection of primary non-MIME message headers are used. This does not decode attachments or unzip archives. Default is `empty`. In this mechanism checks can be made against regular expressions.

For example, postfix can search for any string in an incoming email. In `/etc/postfix/main.cf` set:

    header_checks = regexp: /etc/postfix/headers_checks

Then in `/etc/postfix/header_checks` append regular expressions for what to check for:

    /^(.*)@domain.com/ REJECT    //Match any recipient/sender for a specific domain

The mechanism can also be used to hide a sender's IP and user agent in the `Received` header. This is a privacy concern mostly for sending email with Thunderbird. The received header will contain LAN and WAN IP and info about the email client used. 

Append to `main.cf`:

    smtp_header_checks = regexp:/etc/postfix/smtp_header_checks

Create `/etc/postfix/smtp_header_checks` and append:

    /^Received: .*/     IGNORE
    /^User-Agent: .*/   IGNORE

### Relaying 

Relaying means that a sender with host in domain A, connects to our mailer in domain B to send an email to someone in domain C. A party for spammers. The easiest way to control relaying is to use the `smtpd_recipient_restrictions` directive; If this directive is left undefined, Postfix uses the information given in the directives `permit_mynetworks` and `check_relay_domains`.

## Rule-based mail processing 

With policy services Postfix' behaviour of mail delivery can be fine-tuned. This allows for implementing time-aware grey- and blacklisting of senders and receivers as well as SPF policy checking. 

Policy services are standalone services and connected to Postfix in `/etc/postfix/main.cf` by something like:

    smtpd_recipient_restrictions =
                        ...
      check_policy_service unix:/run/policyd.sock
      check_policy_service inet:127.0.0.1:10040

Placed at the end of the queue reduces load, as only legitimate mails are processed, but place it before the first permit statement to catch all incoming messages. 

## Notifications 

    notify_classes = resource,software,bounce

Postfix reports errors to the postmaster, organised in classes (`notify_classes`). The default is to report only the most serious problems (`resource, software`).
* `bounce` (also implies `2bounce`): Send the postmaster copies of the headers of bounced mail, and send transcripts of SMTP sessions when Postfix rejects mail. The notification is sent to the address specified with the `bounce_notice_recipient` configuration parameter (default: `postmaster`). 
* `2bounce`: Send undeliverable bounced mail to the postmaster. The notification is sent to the address specified with the `2bounce_notice_recipient` configuration parameter (default: `postmaster`). 

## Email loops 

An email loop is an infinite loop phenomenon, resulting from mail servers, scripts, or email clients that generate automatic replies or responses. If one such automatic response triggers another automatic response on the other side, an email loop is created. The process can continue until one mailbox is full or reaches its mail sending limit. Email loops may be caused accidentally or maliciously, causing denial of service (DoS). 

Email loops are not common anymore, due to changes to email software on the client side and the server side, that prevent automatic replies to bounced mail responses.

In Postfix, the maximal number of Received: message headers that is allowed in the primary message headers can be set with `hopcount_limit` (default: 50). A message that exceeds the limit is bounced, in order to stop a mailer loop.

## Reload 

    # postfix reload

## Restart 

    # systemctl restart postfix
    # systemctl status postfix
    # netstat -tlpn


