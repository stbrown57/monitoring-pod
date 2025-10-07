# Rsyslog in Podman Container

Each instance of rsyslog will be run in a container on the host.

# Podman Volume

podman volume create rsyslog.conf

The mount point is /etc/rsyslog.d/

Examle config

```conf
# 10-remote.conf
# Discard all info, notice, and debug messages from all facilities
*.* stop
*.warn action(type="omfwd" Target="SYSLOG_SERVER_IP" Port="514"
Protocol="tcp")
# Forward everything else (that wasn't discarded above) to the central
server
*.* action(type="omfwd" Target="SYSLOG_SERVER_IP" Port="514"
Protocol="tcp")
```

OK, that may get syslog data to the logserver, but Alloy can probably do that directly, however VyOS can not automaticlly forward Surcata logs to Loki since it is not using Alloy. The VyOS instance should be keep as "atomic" as possible, with the distributed utilities. So lets continue to the Rsyslog Syslog server.


# Syslog Server Container

The Rsyslog server will be a part of the "monitoring-pod". 