# Monitoring Pod Developmenet

A Prometheus / Loki / Grafana pod was created from the command line using Podman. The network was a specially created Podman network and persistent data was configured to save to Podman volumes. Alloy was added to collect metrics on the container host system.

Metric collection seems to be working correctly using the Alloy configuration found in the alloy-scenarios repository, however I am not getting journal logs. The demo docker-compose instance seems to collect the journal logs from the container image not the host journal even with the hosts /var/log/journal directory mounted in the container at /var/log/journal with the :z attribute.

Even with the proper configuration of the journal scrape, I may want to use rsyslog for filtering, consolidation and collecting data from devices that are not running Alloy. The VyOS router is an example. It collects and forwards logs with a Telegraf agent, but does not provide the capability to send Suricata logs to the Loki server, or at least I haven'e figured it out.

The test configuration [test-alloy-config](/test-alloy-config) worked.

## Pod "monitoring-pod"

The monitoring-pod was inspected to retrieve the commands used to create it. 

```bash
podman pod inspect monitoring-pod
```

Which returned this relevant data:

```json
          "CreateCommand": [
               "podman",
               "pod",
               "create",
               "--name",
               "monitoring-pod",
               "--network",
               "bfnet"
          ],
```

Quadlet pod file:

```
[Unit]
Description=Application Stack Pod Definition
Wants=network-online.target
After=network-online.target

[Pod]
# The name assigned to the Podman pod
PodName=monitoring-pod

# Configure the pod to use a specific, predefined Podman network.
# This assumes you have an 'app-net.network' Quadlet file or a network named 'app-net' exists.
Network=bfnet
PodmanArgs=--static-ip 192.168.1.16

# Set a label on the pod itself
Label=app.env=production

[Service]
# Ensures the pod is restarted if the entire system reboots or the pod service fails
Restart=always
TimeoutStartSec=90
```

This started the pod, but when I stop the pod all containers were removed. Here is the history of the creations.

### Prometheus

```bash
podman run --pod monitoring-pod -d \
-v /var/lib/containers/storage/volumes/prometheus-config/_data/prometheus.yml:/etc/prometheus/prometheus.yml:z \
-v prometheus.data:/prometheus:z \
--name prometheus \
prom/prometheus "--config.file=/etc/prometheus/prometheus.yml" "--storage.tsdb.path=/prometheus" "--web.enable-remote-write-receiver"
```

### Loki

```bash
podman run --pod monitoring-pod -d \
-v /var/lib/containers/storage/volumes/loki-config/_data/loki-config.yml:/etc/loki/config.yml:z \
--name loki \
grafana/loki
```

### Alloy

```bash
podman run --pod monitoring-pod \
-v alloy.conf:/etc/alloy:z  \
-v alloy.data:/var/lib/alloy/data:z \
--name alloy \
docker.io/grafana/alloy:latest run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.all
```

### Grafana

```bash
podman run --pod monitoring-pod -d --name grafana -v grafana_data:/var/lib/grafana:z docker.io/grafana/grafana
```

### sFlow-RT

```bash
podman run -d --rm --pod monitoring-pod --name sflow-prom sflow/prometheus
```

## Refactor for Pod Static Address

From a Gemini proposed sample:

```bash
# Replace:
# 1. parent=eth0 with your host's interface name.
# 2. --subnet, --gateway, and --ip-range with values from your network.

sudo podman network create \
  --driver=macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.16/28 \
  -o parent=eth0 \
  bfnet
  ```

There is a suggestion to use a Quadlet ".network" file named bfnet.network

```config
[Unit]
Description=Static IP Macvlan Network

[Network]
Driver=macvlan
Options=parent=eth0
Subnet=192.168.1.0/24
IPRange=192.168.1.16/28
Gateway=192.168.1.1
```

#### Prometheus container (Not Working)

```ini
[Container]
ContainerName=prometheus
Image=prom/prometheus
Pod=monitoring-pod
Volume=/var/lib/containers/storage/volumes/prometheus-config/:/etc/prometheus:z
Volume=prometheus.data:/prometheus:z
Exec=--config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --web.enable-remote-write-receiver

[Install]
WantedBy=multi-user.target
```