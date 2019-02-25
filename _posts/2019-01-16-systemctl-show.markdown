---
layout: post
title:  "Use `systemctl show` to output more formated service status"
date:   2019-01-16 19:00:00 +0800
categories: service, automation
---

In automation, we often need to parse output of certain command to get some key information. To get status of a service, we can parse the output of the `systemctl` command. However, output of `systemctl status <service_name>` is more human readable and less machine readable. For example:
```
$ systemctl status cron
● cron.service - Regular background program processing daemon
   Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2019-01-16 10:20:56 CST; 8h ago
     Docs: man:cron(8)
 Main PID: 518 (cron)
    Tasks: 1 (limit: 2320)
   CGroup: /system.slice/cron.service
           └─518 /usr/sbin/cron -f

Jan 16 16:17:01 luvm CRON[1980]: pam_unix(cron:session): session opened for user root by (uid=0)
Jan 16 16:17:01 luvm CRON[1981]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
Jan 16 16:17:01 luvm CRON[1980]: pam_unix(cron:session): session closed for user root
Jan 16 16:38:01 luvm CRON[1989]: pam_unix(cron:session): session opened for user root by (uid=0)
Jan 16 16:38:01 luvm CRON[1989]: pam_unix(cron:session): session closed for user root
Jan 16 17:17:01 luvm CRON[2018]: pam_unix(cron:session): session opened for user root by (uid=0)
Jan 16 17:17:01 luvm CRON[2019]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
Jan 16 17:17:01 luvm CRON[2018]: pam_unix(cron:session): session closed for user root
Jan 16 18:17:01 luvm CRON[2088]: pam_unix(cron:session): session opened for user root by (uid=0)
Jan 16 18:17:01 luvm CRON[2088]: pam_unix(cron:session): session closed for user root
```

It is a little bit difficult to parse service status from this kind of output. To get a more machine readable output, you can use the `systemctl show` command, for example:
```
$ systemctl show cron
Type=simple
Restart=no
NotifyAccess=none
RestartUSec=100ms
TimeoutStartUSec=1min 30s
TimeoutStopUSec=1min 30s
RuntimeMaxUSec=infinity
WatchdogUSec=0
WatchdogTimestamp=Wed 2019-01-16 10:20:56 CST
WatchdogTimestampMonotonic=6486296
PermissionsStartOnly=no
RootDirectoryStartOnly=no
RemainAfterExit=no
GuessMainPID=yes
...
```

By default it outputs all service related details in key=value format. You can use the --property argument to limit the properties to be output. For example:
```
$ systemctl show --property=ActiveState --property=SubState cron
ActiveState=active
SubState=running
```

It would be a trivial task to parse output of this format.
