# PROMTAIL-LOKI-GRAFANA #

chech before install 

docker ps 

docker inspect contID  | grep LogPath

sudo ls -l     (LogPath)/var/lib/docker/containers/a30013de90c17999f0fce486681c893733f9acb685d1c849d53b1aa49b4b8f0f/a30013de90c17999f0fce486681c893733f9acb685d1c849d53b1aa49b4b8f0f-json.log



# install loki
cd /opt
wget https://github.com/grafana/loki/releases/download/v2.9.4/loki-linux-arm64.zip

apt update
apt install zip -y


unzip loki-linux-arm64.zip

chmod +x loki-linux-arm64
mv loki-linux-arm64 /usr/local/bin/loki


loki --version


mkdir -p /var/lib/loki

#loki conf directory
mkdir -p /etc/loki
vi /etc/loki/loki.yaml

auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 72h

#create loki storage directory

mkdir -p /var/lib/loki/chunks
mkdir -p /var/lib/loki/rules

loki -config.file=/etc/loki/loki.yaml

check 
curl http://localhost:3100/ready

# promtail install 

cd /opt
wget https://github.com/grafana/loki/releases/download/v2.9.4/promtail-linux-arm64.zip

apt update
apt install zip -y

unzip promtail-linux-arm64.zip

chmod +x promtail-linux-arm64
mv promtail-linux-arm64 /usr/local/bin/promtail

promtail --version

mkdir -p /etc/promtail

vi /etc/promtail/promtail.yaml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://10.2.1.131:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          instance: nest-node-5
          __path__: /var/lib/docker/containers/*/*-json.log

    pipeline_stages:
      - json:
          expressions:
            log: log
      - regex:
          expression: '/var/lib/docker/containers/(?P<container>[^/]+)/.*'
      - labels:
          container:

promtail -config.file=/etc/promtail/promtail.yaml

check
curl http://localhost:9080/ready


curl http://localhost:3100/metrics | grep promtail
curl -G http://localhost:3100/loki/api/v1/labels

# Grafana install 

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg

#verify
ls -l /etc/apt/keyrings/grafana.gpg

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update

sudo apt install -y grafana

sudo systemctl start grafana-server
sudo systemctl enable grafana-server
systemctl status grafana-server

#interate loki with grafana
go to ------>Administration → Data Sources → Add Data Source



