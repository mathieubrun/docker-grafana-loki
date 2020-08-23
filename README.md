# Lightweight monitoring stack

Based on :

- InfluxDB: metrics persistence
- Grafana: dashboards
- Loki: logs aggregation
- Telegraf: metrics collection
- Promtail: logs collectin

## Log collection with promtail

### Installation

Get the promtail release for your system : [https://github.com/grafana/loki/releases/tag/v1.6.0]() (see below for compilation instructions, if deploying on a Raspberry Pi).

### Configuration

Create the configuration file `/etc/promtail.yaml` and replace LOKI_SERVER with the server IP where loki will be deployed.

```` yaml
server:
  http_listen_port: 9090
  grpc_listen_port: 0

clients:
  - url: http://LOKI_SERVER:3100/api/prom/push

scrape_configs:
  - job_name: journal
    journal:
      max_age: 12h
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels:
        - __journal__systemd_unit
        target_label: unit
      - source_labels:
        - __journal__hostname
        target_label: nodename
      - source_labels:
        - __journal_syslog_identifier
        target_label: syslog_identifier
      - source_labels:
        - __journal_container_name
        target_label: container_name
````

### Running as a service

Create the systemd service file in `/etc/systemd/system/promtail.service`

````
[Unit]
Description=Promtail Loki Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file /etc/promtail.yaml
WorkingDirectory=/usr/local/bin
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
````

Then enable and start the service :

```` sh
sudo systemctl enable promtail.service
sudo systemctl start promtail.service
````

### Docker container logs

For collecting logs of docker containers, the easiest way is to change the default loggin driver to `journald`

```` json
{
    "log-driver": "journald"
}
````

Then restart the docker service using `sudo systemctl restart docker.service`

### Compile from sources

Compile promtail on your raspberry, as it is not yet available with journald support, and not available for armv6. This can take a while on a Pi Zero and may require [expanding swap](https://www.raspberrypi.org/forums/viewtopic.php?t=46472).

```` sh
sudo apt install libsystemd-dev
wget https://golang.org/dl/go1.15.linux-armv6l.tar.gz
tar -zxvf go1.15.linux-armv6l.tar.gz

wget https://github.com/grafana/loki/archive/v1.6.0.tar.gz
tar -zxvf v1.6.0.tar.gz

cd  ~/loki-1.6.0/cmd/promtail
CGO_ENABLED=1 ~/go/bin/go build

rm -rf ~/v1.6.0.tar.gz ~/loki ~/go1.15.linux-armv6l.tar.gz ~/go ~/.cache/go-build
````

## Metrics collection with telegraf

### Installation

```` sh
echo "# Installing telegraf"
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
echo "deb https://repos.influxdata.com/debian buster stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
sudo apt update
sudo apt install -y telegraf
sudo usermod -aG docker telegraf
sudo systemctl restart telegraf
````

### Configuration

Create the configuration file `/etc/telegraf/telegraf.conf` and replace INFLUXDB_SERVER with the server IP where influxdb will be deployed.

````
[global_tags]

[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  hostname = ""
  omit_hostname = false

[[outputs.influxdb]]
  urls = ["http://INFLUXDB_SERVER:8086"]
  database = "telegraf"

[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]

[[inputs.diskio]]

[[inputs.kernel]]

[[inputs.mem]]

[[inputs.processes]]

[[inputs.swap]]

[[inputs.system]]
````

## Logs and metrics aggregation

Clone or download this repository, create folders and run docker-compose stack:

```` sh
echo "# Creating folders"
mkdir -p data/loki
sudo chown -R 1000:1000 data/loki
mkdir -p data/grafana
sudo chown -R 1000:1000 data/grafana
echo "# Running monitoring stack"
docker-compose up -d
````

## Todo

- networks
- more docs
- backups
