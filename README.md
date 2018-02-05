# Mail configuration

This repository contains the minimal configuration required for a useful
Postfix, Dovecot, and OpenDKIM setup.  The configuration files are offered in
a Jinja2 template format.

Goals:

- Reasonable defaults
- As secure as possible
- Virtual mailboxes
- Support for acmetool (hlandau/acme)

Dependencies:
- Python
- Jinja2

**This is a work in progress. TODO:**

- Improve deploy script
- Change password digest to Argon
- Implement DKIM keygen helper


## Usage

Copy the default configuration and edit them as required:

    cp -r config.dist config

Run the deploy script:

    ./bin/deploy


## Meta configuration files

* `domains`: list of accepted domains, the first domain is considered the
  primary domain (e.g. used in bounce messages)
* `aliases`: contains aliases that let you map the user part of an email address
  to a different user.
* `virtual`: like aliases, but also allows you to set up domain-specific
  aliases
* `dnsbl`: list of DNSBL hosts to consult, optional
* `passwd`: contains the list of users that can use the mail server, in Dovecot
  passdb format


## Managing users

TODO: argon2

...


## DKIM


### Rotating keys

Determine the name of the selector you will use next (e.g. `dkim$date`).

Generate new keys:

    ./bin/dkim-gen-keys


Publish the public keys using your DNS tools of choice.

Rotate the keys:

    echo dkim$date > config/dkim_selector
    ./bin/deploy
    service opendkim reload
