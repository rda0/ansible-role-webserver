webserver
=========

This role deploys a webserver with:

- `apache2`
- `letsencrypt`

The two most important Ansible files to understand are:

- `roles-servers/webserver/defaults/main.yml`
- `roles-servers/webserver/tasks/apache2/vhosts.yml`

nginx
-----

To deploy `nginx` instead of `apache2`, add the following in the inventory:

```yaml
include_webserver_apache2: False
include_webserver_nginx: True
webserver_tls_terminator: nginx
```

VHost deployment
----------------

By default this role only creates the default VHost, which is reserved and restricted for ISG internal use only.

If the following is configured, it will also deploy VHosts for services and websites:

```yaml
include_webserver_vhosts: True
```

The task `set fact _vhosts` in `roles-servers/webserver/tasks/apache2/vhosts.yml` sets a lot of default values  
only very few configuration options are required to create a VHost, namely that is the VHosts name:

```yaml
webserver_vhosts:
  - { name: vhost1 }
  - { name: vhost2 }
```

Just add any parameters that deviate from the default value.

fifo2syslog
-----------

The systemd service file `/etc/systemd/system/fifo2syslog.service` starts `fifo2syslog`, which in turn starts `apache_syslog_auth`. Note that apache will hang at start if the `/var/log/apache2/lastuse.fifo` is missing or the `fifo2syslog` service not running.
