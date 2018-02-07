# Mail configuration

This repository contains the minimal configuration required for a useful
Postfix, Dovecot, and OpenDKIM setup.  The configuration files are offered in
a Jinja2 template format, and a tool to build complete configuration files.
Or just use the templates as inspiration for your own custom setup!

Goals:
- Reasonable defaults
- Works on Alpine Linux out of the box
- As secure as possible
- Virtual mailboxes
- Support for acmetool (hlandau/acme)

Script dependencies:
- Python
- Jinja2

**This is a work in progress. TODO:**

- Improve deploy script
- Change password digest to Argon (needs Dovecot v2.3+)
- Test Dovecot sieve
- Implement DKIM keygen helper
- Make OpenDKIM optional?


## Usage

Install `postfix`, `dovecot`, and `opendkim` via your package manager of choice:

    apk add opendkim postfix dovecot

Create the directory where mail will be stored:

    install -o vmail -g vpopmail -m 700 -d /var/mail

Copy the default configuration and edit it as required:

    cp -r config.dist config

The `config` directory contains files that are used to build the configuration.
Each filename represents a variable that can be used in the template.

* `domains`: list of accepted domains, the first domain is considered the
  primary domain (e.g. used in bounce messages)
* `aliases`: contains aliases that let you map the user part of an email address
  to a different user.
* `virtual`: like aliases, but also allows you to set up domain-specific
  aliases
* `dnsbl`: list of DNSBL hosts to consult, optional
* `passwd`: contains the list of users that can use the mail server, in Dovecot
  passdb format

Run the deploy script:

    ./bin/deploy


## Managing users

Users and passwords are stored in `/etc/dovecot/passwd` in a format similar to
`/etc/passwd`.

XXX: there is an example script (`bin/mail-passwd`) for creating password file
entries.
Example: `echo password | sudo ./bin/mail-passwd /etc/dovecot/passwd user`


## DKIM

With DKIM, message headers are signed so that the recipient can verify that
the domain listed as sender was indeed the originating domain,
even when an email is forwarded.
Having a valid DKIM signature usually reduces the likelihood of your message
being marked as spam.


### Generating or rotating keys

Determine the name of the selector you will use next (e.g. `dkim$date`).

Generate new keys:

    ./bin/dkim-gen-keys


Publish the public keys using your DNS tools of choice.

Rotate the keys:

    echo dkim$date > config/dkim_selector
    ./bin/deploy
    service opendkim reload


# Implementation details

This setup uses a virtual mailbox setup. That means the mailboxes do not
necessarily relate to accounts available on the server. It also supports adding
an arbitrary number of domains for which you will be able to send and receive
email.
This means the domain of the recipient is ignored when determining the
mailbox.  That also means users login with just their username, not an email
address.


## Postfix

The Postfix setup is fairly straight forward. Remote mail can be delivered on
port 25, and users must submit emails to be sent on port 587 (nb: Postfix
delegates user authentication to Dovecot).
Before sending emails, the email is passed through OpenDKIM for signing.

Email can be delivered both via IPv4 and IPv6.
You should publish the FQDN of the server as an MX record on the domains you
specified in your configuration.
Also set up SPF records (e.g. `v=spf1 +mx +a:$fqdn -all` -- don't forget to
replace `$fqdn`) for each domain.
DKIM records must also be published.

Postfix supports optional STARTTLS for receiving mail (both submission and
delivery) and for sending email. Currently only submission *requires* STARTTLS,
since in every other case we cannot enforce that the other party supports this.
(Sadly, requiring STARTTLS everywhere is a good recipe for not being able to
receive mail from a large number of senders. RFC 2487 also prohibits this for
interoperability.  TODO: maintain a list of compliant domains.)


## Dovecot

Dovecot receives emails from Postfix over LMTP.
Users can retrieve their email using POP3 (TLS only, port 995) or IMAP (TLS
only, port 993).
