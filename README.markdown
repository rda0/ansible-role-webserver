webserver
=========

This role deploys a webserver with:

- `apache2`
- `letsencrypt`

fifo2syslog
-----------

The systemd service file `/etc/systemd/system/fifo2syslog.service` starts `fifo2syslog`, which in turn starts `apache_syslog_auth`. Note that apache will hang at start if the `/var/log/apache2/lastuse.fifo` is missing or the `fifo2syslog` service not running.
