webserver
=========

This role deploys a webserver and consists of the following components:

- `apache2` (default enabled)
- `nginx`
- `letsencrypt` (default enabled)

The two most important Ansible files to understand are:

- `defaults/main.yml`
- `tasks/apache2/vhosts.yml`

letsencrypt
-----------

By default a certificate is created using the hostname only and is named `le0`.
To add an additional service domain, use:

```yaml
webserver_letsencrypt_aliases:
  - domains:
      - '{{ ansible_fqdn }}'
      - example.phys.ethz.ch
```

Multiple and custom named certificates can be created:

```yaml
webserver_letsencrypt_aliases:
  - name: le-default
    domains:
      - '{{ ansible_fqdn }}'
  - name: le-customer1
    domains:
      - '{{ ansible_fqdn }}'
      - customer1.phys.ethz.ch
webserver_site_config_default_vhost_default_https_enablessl: Use EnableSSLCert le-default
```

### letsencrypt staging

To use the [Let's Encrypt staging environment](https://letsencrypt.org/docs/staging-environment/) to avoid hitting the rate limit set `webserver_letsencrypt_use_staging: True`

nginx
-----

To deploy `nginx` instead of `apache2`, add the following in the inventory:

```yaml
include_webserver_apache2: False
include_webserver_nginx: True
webserver_tls_terminator: nginx
```

Example vhost config:

```yaml
webserver_vhosts:
  - name: example
    server_name: example.phys.ethz.ch
    backend_config: |
      upstream appservers {
          server 127.0.0.1:8080;
          server 127.0.0.1:8081;
      }
    config: |
      root /var/www/html;
      index index.html;
      location / {
          try_files $uri $uri/ =404;
      }
      location /app {
          proxy_pass http://appservers;
      }
```

apache2
-------

Configures an apache2 http server.

### common options

```yaml
webserver_ldap: True                        # enable ldap auth
webserver_packages_custom: []               # extra packages to install
webserver_modules_custom: []                # extra modules to enable
webserver_site_content_default_description: Example webserver
include_webserver_phpfpm: True              # enable phpfpm
include_webserver_website: True             # enable deployment of websites
include_webserver_website_db: True          # enable deployment of databases for websites
include_webserver_website_wordpress: True   # enable deployment of wordpress websites
webserver_phpfpm_pools: []                  # list of phpfpm pools to create
include_webserver_sites_include: True       # enable deployment of /etc/apache2/sites-include
```

For details and all other settings refer to `defaults/main.yml`.

### phpfpm pools

Pool names must be identical to the vhost names, example:

```yaml
webserver_phpfpm_pools:
  - name: vhost1
  - name: vhost2
    php_options: |
      php_value[max_input_vars] = 5000
```

### VHost deployment

By default this role only creates the default VHost, which is reserved and restricted for ISG internal use only.

If the following is configured (the default), it will also deploy VHosts for services and websites:

```yaml
include_webserver_vhosts: True
```

The task `set fact _vhosts` in `roles-servers/webserver/tasks/apache2/vhosts.yml` sets a lot of default values
only very few configuration options are required to create a VHost, namely that is the VHosts `name`:

```yaml
webserver_vhosts:
  - { name: vhost1 }
  - { name: vhost2 }
```

Just add any parameters that deviate from the default value.

To filter vhosts and apply vhost settings to a single vhost or a set of vhosts for the current run,
use the `vhostfilter` setting, example:

```bash
-e vhostfilter=vhost1
```

Commonly used options are:

```yaml
webserver_vhosts:
  - name: example                                       # the name (required)
    type: share                                         # `server` (default), in `/var/www`, user `www-data`
                                                        # `share` in `/home`, user is same as `name`
                                                        # `redirect` for empty sites only redirecting externally
    server_name: example.phys.ethz.ch                   # main domain name
    server_alias: foo.ch bar.org                        # list of domain aliases
    home_user_owned: True                               # allow home to be accessible via ssh (disables group access)
    http: {}                                            # http vhost settings (should generally not be used)
    https:                                              # https settings
      cert: le-customer1                                # certificate to use
      redirect: 'https baz.com to/path'                 # redirect somewhere else
      require: grant_share_all_allow_override_none      # access template from `webserver_site_config_common_requires`
      config: 'DirectoryIndex disabled'                 # custom config to include at the end of the vhost
      fcgid:
        extensions: ['cgi','php','py']                  # list of cgi extensions to enable cgi execution for
        options:                                        # options for cgi (php has default options enabled)
          php: ['foo=1','bar=2']                        # override php options
      wsgi:
        app: example.wsgi                               # wsgi entry point in public/
        venv: True                                      # deploy a venv in private/venv
        static: ['static','other/static']               # list of paths to serve statically
        processes: 4                                    # default 2
        threads: 20                                     # default 5
      includes: ['foo','bar']                           # *.conf files to optionally include from sites-include dir
                                                        # by default a file named like `name` is included optionally
    website:                                            # website specific settings
      type: wordpress                                   # deploy wordpress (needs secret)
      db: True                                          # deploy a database (needs secret)
    secret: "{{ secret_variable }}"                     # secret to use for website and database passwords
```

For details and all other settings refer to `tasks/apache2/vhosts.yml`.

### wsgi venv

The venv is deployed according to `private/requirements.txt`, this is the file that should be maintained by the user.
If a `private/freeze.txt` is found, this will take precedence over `private/requirements.txt`.
After generating the venv, a `private/freeze.txt` will be generated (for later exact reproducibility).

To regenerate the venv use `-e '{"webserver_wsgi_venv_reinit": True}'`. In addition you may want to manually delete the `freeze.txt` beforehand.

In rare exceptions you may need to enable access to system-wide Python packages from inside the venv using the `venv_system_pkgs: True` flag. This can be useful if the required packages have no wheels, are non-trivial to build from source, but readily available as Debian package. In general this setting should be avoided as the webshare may be impacted by Debian package and release upgrades.

### fifo2syslog

The systemd service file `/etc/systemd/system/fifo2syslog.service` starts `fifo2syslog`, which in turn starts `apache_syslog_auth`. Note that apache will hang at start if the `/var/log/apache2/lastuse.fifo` is missing or the `fifo2syslog` service not running.

### authorized keys

To access shares via ssh, `sshd` needs to be configured with `StrictModes no` and ssh keys must be used.
The users need to connect as the webshare user in the form of `ssh wwwshare@server`.

Keys per user must be defined in `webserver_share_authorized_keys` or a global vars file in the following format:

```yaml
webserver_share_authorized_keys:
  user1: |
    # User1 <user1@email>
    ssh-ed25519 pub-key user1@host
  user2: |
    # User2 <user2@email>
    ssh-rsa pub-key user2@host
```

and can be deployed on a vhost as follows:

```yaml
webserver_vhosts:
  - type: share
    authorized_keys:
      - user1
      - user2
```

Global variables with key definitions can be included with (example):

```yaml
webserver_share_authorized_keys_vars:
  - ssh_authorized_keys_admin
  - ssh_authorized_keys_other
```

To automatically fill a facts variable `webserver_admin_keys` with a list of admin user names
gathered from a specific list or `webserver_share_authorized_keys_vars` variable use:

```yaml
webserver_admin_keys_var_name: ssh_authorized_keys_admin_list
```

Which can be used in vhosts as follows:

```yaml
webserver_vhosts:
  - type: share
    authorized_keys: "{{ webserver_admin_keys }}"
```

To protect the users of a multi-user webshare against forwarded ssh-agent hijacking attacks,
ssh-agent forwarding is disabled by default for shares with more than one ssh user.
This can be controlled via:

```yaml
webserver_share_authorized_keys_multiuser_restrict_options: 'restrict,pty'
```

To override this on a vhost use `vhost.authorized_keys_multiuser_options`.

Migration
---------

To migrate a host to this role, these instructions could be helpful:

### 1. letsencrypt

```bash
cd /etc/apache2/conf-available
unlink 010-well-known.conf
scp phd-apache:/etc/apache2/conf-available/010-well-known.conf
```

Check where the certs are included:

```bash
grep -Ri SSLCertificate /etc/apache2
```

Change all to:

```
SSLCertificateFile      /opt/letsencrypt/le0.rsa.crt
SSLCertificateKeyFile   /opt/letsencrypt/le0.rsa.key
```

Prepare ansible inventory:

```yaml
include_webserver_apache2: False
webserver_letsencrypt_aliases:
  - domains:
      - '{{ ansible_fqdn }}'
      - service.phys.ethz.ch
      - ...
```

Make some cleanup:

```bash
cd /opt
mv letsencrypt letsencrypt_bak
apt purge acme-tiny
userdel letsencrypt
```

Push the `webserver` role. Cleanup `letsencrypt_bak`.
