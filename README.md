# ansible-freebsd-mta

[Ansible][ansible] playbook to configure [FreeBSD][freebsd] 12.x server
to work as [MTA][mta] with support for multiple domains to serve mail for.

Software in use:

 - basic [FreeBSD][freebsd] 12.x installation
 - [postfix][postfix] for mail transfer
 - [dovecot][dovecot] for mail delivery using IMAPS
 - [rspamd][rspamd] for spam filtering
 - [redis][redis] as a local storage used by [rspamd][rspamd]
 - [acme.sh][acme.sh] to issue certificate using [Let's Encrypt][letsencrypt]
     CA for secure communication with server (certificate will be issued using
     [acme.sh DNS API][acme.sh-dnsapi])
 - [sqlite][sqlite] as virtual mail users storage
 - [unbound][unbound] as a local caching DNS resolver


## Sources of inspiration

- https://www.c0ffee.net/blog/mail-server-guide/
- https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/
- https://thomas-leister.de/en/mailserver-debian-stretch/
- https://blog.crashed.org/letsencrypt-in-freebsd-org/


## Usage

- prepare local host
- clone repository
- prepare `vault_pass` file
- prepare `secrets.yml` file
- prepare `config.yml` file
- prepare target host
- prepare `hosts` file
- run playbook
- update DNS MX record
- update DNS with provided SPF, DKIM, DMARC records
- update DNS provider PTR record for mail server host


### Prepare local host

- install git
- install ansible
- install python-netaddr
- install python-passlib


### Clone repository

```shell
$ git clone https://github.com/ysegorov/ansible-freebsd-mta.git
$ cd ansible-freebsd-mta
```


### Prepare `vault_pass` file

> DO NOT commit this file to repository!
> It's ignored by default in `.gitignore`.

`vault_pass` is a file to be used by [Ansible Vault][ansible-vault] which is
used to store passwords and other private data encrypted.

This file can be either **an executable to return password** to encrypt data or
**a plain text file containing password**:

- for the former case the file might look like this (actual password is stored
    encrypted using [passwordstore][passwordstore] which in turn uses
    [GPG][gpg] under the hood):

    ```
    #!/usr/bin/env bash

    pass ansible-freebsd-mta/vault-pass
    ```

    Keep `vault_pass` file permissions restricted:

    ```shell
    $ chmod 700 vault_pass
    ```

- for the latter case the file might look like this:

    ```
    some-super-secret-password
    ```

    Keep `vault_pass` file permissions restricted:

    ```shell
    $ chmod 600 vault_pass
    ```


### Prepare `secrets.yml` file

> DO NOT commit this file to repository!
> It's ignored by default in `.gitignore`.

`secrets.yml` is a file to store mail users passwords and other sensitive data
encrypted.

It will be loaded by [ansible][ansible] when running playbook.

First, create an empty `secrets.yml` file and encrypt it using `ansible-vault`:

```shell
$ touch secrets.yml
$ ansible-vault encrypt secrets.yml
```

Then edit it using `ansible-vault`:

```shell
$ ansible-vault edit secrets.yml
```

Keep in mind that:

- this file must contain [YAML][yaml] formatted data suitable for
    [ansible][ansible]
- you must **save and close** your editor to apply changes (file will be
    re-encrypted by `ansible-vault` on close)

___Example___:

```yaml
---

# Linode api key used by acme.sh to add domain record for domain validation
vault_linode_api_key: API-KEY-VALUE

# domain1.tld users
vault_john_at_domain1_tld_password: john-password
vault_foo_at_domain1_tld_password: foo-password

# domain2.tld users
vault_baz_at_domain2_tld_password: baz-password
vault_bar_at_domain2_tld_password: bar-password
```


### Prepare `config.yml` file

> DO NOT commit this file to repository!
> It's ignored by default in `.gitignore`.

`config.yml` is a file to configure playbook providing:

- `hostname` for the server
- `postmaster` email
- `acme.sh` configuration values
- `vmail` mail server domains/users to serve

___Example___:

```yaml
---

hostname: mta.domain1.tld
postmaster: john@domain1.tld

unbound_use_cloudflare_dns: yes

acme_dns: linode_v4
acme_dnssleep: 1800
acme_env:
  - key: LINODE_V4_API_KEY
    value: "{{ vault_linode_api_key }}"

vmail:
  - domain: mta.domain1.tld
    aliases:
      - email: root@mta.domain1.tld
        dest: john+root@domain1.tld
      - email: vmail@mta.domain1.tld
        dest: john+vmail@domain1.tld
      # this is 'catch all mail' address syntax
      - email: "@mta.domain1.tld"
        dest: john+mta@domain1.tld
  - domain: domain1.tld
    aliases:
      - email: adv@domain1.tld
        dest: john@domain1.tld,foo@domain1.tld
    users:
      - email: john@domain1.tld
        password: "{{ vault_john_at_domain1_tld_password }}"
      - email: foo@domain1.tld
        password: "{{ vault_foo_at_domain1_tld_password }}"
  - domain: domain2.tld
    users:
      - email: baz@domain2.tld
        password: "{{ vault_baz_at_domain2_tld_password }}"
      - email: bar@domain2.tld
        password: "{{ vault_bar_at_domain2_tld_password }}"
```

Configuration parameters:

- `hostname` - proper DNS name for the server (defaults to the value
    automatically discovered by [ansible][ansible])
- `postmaster` - real user email address to receive occasional admin mails or
    abuse reports
- `unbound_use_cloudflare_dns` - `yes`/`no` boolean indicating that local
    caching DNS resolver must use **CloudFlare DNS** as an upstream resolver
    (see [1.1.1.1][1.1.1.1] for details)
- `acme_dns` - DNS provider name to be used by [acme.sh][acme.sh] to
    automatically issue certificate to secure communications (see [acme.sh DNS
    API][acme.sh-dnsapi] for a list of supported providers)
- `acme_dnssleep` - number of seconds to wait for DNS changes to take effect
    (needed for [Linode][linode] and some others to publish changes)
- `acme_env` - list of `key`/`value` pairs needed to provide [acme.sh][acme.sh]
    with credentials to access DNS provider API (see [acme.sh DNS
    API][acme.sh-dnsapi] for details)
- `vmail` - list of mail server domains/aliases/users to serve for (see example
    above for syntax)

**You must configure mail server to receive email for `hostname` to
avoid possible issues with local mail server email delivery** (see example
above).

Notes on email address syntax:

- having defined `foo@domain1.tld` user will be able to receive messages for
    addresses like `foo+tag@domain1.tld` - feature sometimes called *tagged*
    mailbox
- you can define per domain *catch-all* address like `@domain1.tld`
    to receive all messages destined to unknown/undefined mailboxes
- you can define multiple destinations for an alias - use comma to separate
    several target addresses


### Prepare target host

Install [FreeBSD][freebsd] taking into account following notes:

- don't enable `sendmail`
- don't enable `local_unbound`
- properly configure `sshd` (use custom port, disable password-based login)
- consider using separate block storage to hold emails (mount it to
    `/var/vmail` mountpoint)

Reference manual on how to install [FreeBSD][freebsd] on [Linode][linode] is
located [here][linode-freebsd].


### Prepare `hosts` file

> DO NOT commit this file to repository!
> It's ignored by default in `.gitignore`.

`hosts` file is an *inventory* file in terms of [ansible][ansible].

It must contain mail server host **ip address** and **ssh port** to use
to connect to the server.

___Example___:
```
mta.domain1.tld ansible_host=AAA.BBB.CCC.DDD ansible_port=xxxx

[mta]
mta.domain1.tld
```

**You must add hostname to `[mta]` group for playbook to work** (see example).


### Run playbook

Use provided `play` script to run playbook:

```sh
$ ./play
```

You can update `config.yml`/`secrets.yml` later and update mail users
database using following command:

```sh
$ ./play -t vmail
```


### Update DNS

#### MX record

Please add proper **MX** record for each domain mail server will serve for
using `hostname` value.


#### SPF, DKIM and DMARC records

Please check `dns/` subfolder which will contain per-domain text files with DNS
records to be added to correspondent DNS zone.


#### PTR record

Please configure proper **PTR** record for mail server (see [Linode Reverse
DNS][linode-reverse-dns] manual for reference).


## Mail program configuration parameters


### Incoming mail server

| parameter             | value                          |
|-----------------------|--------------------------------|
| server type           | IMAP                           |
| server name           | `hostname` value               |
| port                  | 993                            |
| connection security   | SSL/TLS                        |
| authentication method | normal password                |
| username/password     | per `config.yml`/`secrets.yml` |


### Outgoing mail server

| parameter             | value                          |
|-----------------------|--------------------------------|
| server type           | SMTP                           |
| server name           | `hostname` value               |
| port                  | 587                            |
| connection security   | STARTTLS                       |
| authentication method | normal password                |
| username/password     | per `config.yml`/`secrets.yml` |


## How to contribute

Fork repository, make some changes and create pull request.


## License

Licensed under terms of **BSD 3-Clause License** (see LICENSE.md).



[ansible]: https://www.ansible.com
[ansible-vault]: https://docs.ansible.com/ansible/latest/user_guide/vault.html
[freebsd]: https://www.freebsd.org
[mta]: https://en.wikipedia.org/wiki/Message_transfer_agent
[postfix]: http://www.postfix.org/
[dovecot]: https://dovecot.org/
[rspamd]: https://rspamd.com/
[redis]: https://redis.io/
[acme.sh]: https://acme.sh/
[acme.sh-dnsapi]: https://github.com/Neilpang/acme.sh/wiki/dnsapi
[sqlite]: https://sqlite.org/index.html
[unbound]: https://nlnetlabs.nl/projects/unbound/about/
[letsencrypt]: https://letsencrypt.org/
[passwordstore]: https://www.passwordstore.org/
[gpg]: https://gnupg.org/
[yaml]: https://yaml.org/
[1.1.1.1]: https://1.1.1.1/dns/
[linode]: https://www.linode.com
[linode-freebsd]: https://www.linode.com/docs/tools-reference/custom-kernels-distros/install-freebsd-on-linode/
[linode-reverse-dns]: https://www.linode.com/docs/networking/dns/configure-your-linode-for-reverse-dns/
