The `sshd` role creates the following two files on the server, setting secure defaults.

* `/etc/ssh/sshd_config`
* `/etc/ssh/ssh_config`

## Full configuration

To keep the files as simple as possible, options are omitted if their system defaults are secure and broadly applicable. You may see the full and active configuration by running the following commands on your server.

* SSH server (`sshd_config`): `sshd -T`
* SSH client (`ssh_config`): `ssh -G example.com`

There are [resources](#resources) for understanding each option.

## Customize via variables

You may redefine any variable found in `templates/sshd_config.j2` or `templates/ssh_config.j2`. The default settings are viewable in `defaults/main.yml`. To override a setting, you could redefine your chosen variable in a file such as `group_vars/all/main.yml` or `group_vars/all/security.yml`. If you don't find a variable for the setting you need to change, you may need to [customize via child templates](#customize-via-child-templates).

### Basic variable override

Suppose you want your SSH server to `AcceptEnv`, whereas the Trellis default does not accept any env variables. You could find the relevant variable name from `templates/sshd_config.j2` or from `defaults/main.yml`, then redefine that variable in `group_vars/all/main.yml`. In this example, the relevant variable is `sshd_accept_env` and is formatted as a list.

```
# group_vars/all/main.yml

sshd_accept_env:
  - LANG
  - LC_*
```

You may notice that `templates/ssh_config.j2` references some `ssh_<varname>` variables that are not included in `defaults/main.yml` and that default to a `sshd_<varname>` variable. Here is an example:
```
AddressFamily {{ ssh_address_family | default(sshd_address_family) }}
```
This pattern spares `defaults/main.yml` from having repetitious `ssh` and `sshd` definitions for all settings. You may still define custom values for any `ssh_<varname>` in your `group_vars` files.

### `Ciphers`, `KexAlgorithms`, and `MACs`

The variables for `Ciphers`, `KexAlgorithms`, and `MACs` are split into `<varname>_default` and `<varname>_extra` (e.g., `sshd_macs_default` and `sshd_macs_extra`). The `<varname>_default` contains a list you will probably not need to change. You may use `<varname>_extra` to supplement the default lists. SSH connections involving older systems may require some of the less secure options below.

```
# group_vars/all/security.yml

# Allow CBC mode ciphers (less secure)
sshd_ciphers_extra:
  - aes256-cbc
  - aes192-cbc
  - aes128-cbc

# Accommodate older systems by allowing weaker kex algorithms (less secure)
sshd_kex_algorithms_extra:
  - diffie-hellman-group14-sha1
  - diffie-hellman-group-exchange-sha1
  - diffie-hellman-group1-sha1

# Accommodate older systems by allowing weaker MACs (less secure)
sshd_macs_extra:
  - umac-128@openssh.com
  - hmac-sha1
```

## Customize via child templates

If you can't [customize via variables](#customize-via-variables) because the template doesn't include a variable for the setting you want to change, first check the [full configuration](#full-configuration) to verify that the default in effect is not what you want. If you need to make a change, you may create a child template to override the default template.

Create your child templates following the [Jinja template inheritance](http://jinja.pocoo.org/docs/latest/templates/#template-inheritance) docs and the guidelines below.


### Designate a child template

Use the `sshd_config` and `ssh_config` variables to inform Trellis of the child templates you have created. Below is an example of designating child templates in a new `templates` directory in your Trellis project root (e.g., next to the `server.yml` playbook).

```
# group_vars/all/main.yml

sshd_config: "{{ playbook_dir }}/templates/sshd_config.j2"
ssh_config: "{{ playbook_dir }}/templates/ssh_config.j2"
```

### Create a child template

Create your child templates at the paths you designated in the `sshd_config` and `ssh_config` variables described above. [Child templates](http://jinja.pocoo.org/docs/latest/templates/#child-template) must include two elements:

* an `{% extends 'base_template' %}` statement
* one or more `{% block block_name %}` blocks

The path for your base template – referenced in your `extends` statement – must be relative to the `server.yml` playbook (i.e., relative to the Trellis root directory). See the examples below.

Here is an example child template that adds some sftp settings to the end of the `sshd_config`.

```
# templates/sshd_config.j2

{% extends 'roles/sshd/templates/sshd_config.j2' %}

{% block main %}
{{ super() }}
Match Group sftponly
AllowAgentForwarding no
ChrootDirectory /home/%u
ForceCommand internal-sftp
PermitRootLogin no
{%- endblock %}
```
The [`{{ super() }}`](http://jinja.pocoo.org/docs/latest/templates/#super-blocks) Jinja2 function returns the original block content from the base template, and can be omitted if you don't want to include the original content.

Here is an example child template that adds host-specific SSH options at the beginning of `ssh_config`.

```
# templates/ssh_config.j2

{% extends 'roles/sshd/templates/ssh_config.j2' %}

{% block main %}
# Host-specific configuration
Host example.com example2.com
	Port 2222
	ForwardAgent yes

# Global defaults for all Hosts
{{ super() }}
{%- endblock %}
```

## Troubleshooting

In case you have trouble with SSH connections to your server after running this role, here are a few tips.

### Host key change

This role will most likely cause your SSH server to switch to a more secure host key than what your local machine's `known_hosts` has recorded for the host. The next time you attempt an SSH connection, you will probably see a warning like one of the examples below. Run `ssh-keygen -R 12.34.56.78` (edit command with your real IP or host name) on your local machine to clear the old host key out of your `known_hosts`. Then try your SSH connection again.

Example 1

```
TASK [setup] *******************************************************************
System info:
  Ansible 2.2.1.0; Darwin
  Trellis at "Add `apt_packages_custom` to customize Apt packages"
---------------------------------------------------
SSH Error: data could not be sent to the remote host. Make sure this host can
be reached over ssh
fatal: [xxx.xxx.xxx.xxx]: UNREACHABLE! => {"changed": false, "unreachable": true}
    to retry, use: --limit @/Users/yourname/sites/example.com/trellis/deploy.retry
```

Example 2

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:lv86hFykjn8pnOWE2WDWJo8Mzf6FTDMx/yWXOqzK5PU.
```

### `git clone` or `composer install` hangs or fails during deploy

This role may cause your server's SSH client to request stronger host keys from hosts of git repos or composer packages during deploys. This could create the [host-key-change](#host-key-change) problem, but this time on your server instead of your local machine. Follow the same remediation steps, but on the server.

Similarly, this role may cause your server's SSH client to require stronger [ciphers, kex algorithms, and MACs](#ciphers-kexalgorithms-and-macs) than previously. If your `git clone` or `composer install` connections involve older systems that do not support the stronger protocols, you may need to add more options to `ssh_ciphers_extra`, `ssh_kex_algorithms_extra`, or `ssh_macs_extra`.

### Verbose output

SSH connection issues are often difficult to resolve without verbose output. Use the `-vvvv` option with your `ansible-playbook` command:

```
ansible-playbook server.yml -e env=production -vvvv
```

You may also use `-v`, `-vv`, and `-vvv` with manual SSH connections:

```
ssh -v root@12.34.56.78
```

### Manual SSH

If your `ansible-playbook` command is failing its SSH connection, it can be helpful to try a manual SSH connection to narrow down the problem. If manual SSH fails, try again with `-v` for [verbose output](#verbose-output).

```
ssh -v root@12.34.56.78
```

### `Ciphers`, `KexAlgorithms`, or `MACs`

This role will most likely cause your SSH server to discontinue using some older and weaker protocols. If your connections involve older systems that do not support the stronger protocols configured by this role, see [`Ciphers`, `KexAlgorithms`, and `MACs`](#ciphers-kexalgorithms-and-macs) for how to add back in any protocols you need.

## Resources

* Ubuntu manpage for [sshd_config](http://manpages.ubuntu.com/manpages/xenial/en/man5/sshd_config.5.html)
* Ubuntu manpage for [ssh_config](http://manpages.ubuntu.com/manpages/xenial/en/man5/ssh_config.5.html)
* stribika's [Secure Secure Shell](https://stribika.github.io/2015/01/04/secure-secure-shell.html) post
* MozillaWiki's [security guidelines for OpenSSH](https://wiki.mozilla.org/Security/Guidelines/OpenSSH)
* bettercrypto.org's [Applied Crypto Hardening](https://bettercrypto.org/static/applied-crypto-hardening.pdf)

## Attribution

Many thanks to [nickjj](https://github.com/nickjj/) for creating the [original version](https://github.com/nickjj/ansible-sshd/) of this role.
