# Cài đặt Node Exporter (trên các target node)
- Tạo user và group `node-exporter`
```
groupadd node-exporter && useradd -s /sbin/nologin --system -g node-exporter node-exporter
```

- Trên 3 server `server1`, `server2`, `server3`, cài đặt node_exporter
```
mkdir -p /tmp/node-exporter/ && cd /tmp/node-exporter/
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xzvf node_exporter*.tar.gz
mv node_exporter-*.linux-amd64/node_exporter /usr/local/bin/
chown node-exporter: /usr/local/bin/node_exporter
```

- Kiểm tra version của node_exporter
```
node_exporter --version
```

- Tạo Systemd Service cho node_exporter 
```
cat << EOF > /etc/systemd/system/node-exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node-exporter
Group=node-exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.systemd --web.config.file=/tmp/node-exporter/web.yml

[Install]
WantedBy=multi-user.target
EOF
```

- Bảo mật cho Node Exporter API bằng basic authentication
```
cat <<EOF > /tmp/prometheus/web.yml
basic_auth_users:
  admin: '$2a$12$noyKAVjH4kV9JUwRVbki7.4PJxz2rhJ1kZrac8uaJ75Y0cm3lYz46'
EOF
```

- Start node_exporter
```
systemctl daemon-reload
systemctl enable node-exporter.service
systemctl start node-exporter.service
systemctl status node-exporter.service
```

# Cài đặt Prometheus
- Tạo `prometheus` user and group
```
sudo groupadd prometheus && sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```

- Tạo folder lưu Time-Series Database của Prometheus
```
mkdir /var/lib/prometheus
chown -R prometheus: /var/lib/prometheus
```

- Tạo folder chứa Prometheus Rules
```
mkdir -p /etc/prometheus/rules.d && sudo mkdir -p /etc/prometheus/files_sd
chown -R prometheus: /etc/prometheus/rules.d &&  chown -R prometheus: /etc/prometheus/files_sd
chmod -R 775 /etc/prometheus/rules.d && chmod -R 775 /etc/prometheus/files_sd
```

- Tạo folder chứa file cài đặt Prometheus và giải nén
```
mkdir -p /tmp/prometheus && cd /tmp/prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.55.0/prometheus-2.55.0.linux-amd64.tar.gz
tar -zxvf prometheus*.tar.gz
```

- Chuyển binary file `prometheus` và `promtool` vào `/usr/local/bin/` để có thể chạy 2 lệnh trên
```
sudo mv prometheus promtool /usr/local/bin/
```

- Setup Prometheus 
```
mv prometheus.yml /etc/prometheus/prometheus.yml
mv consoles/ console_libraries/ /etc/prometheus/
```

- Bảo mật Prometheus API và Prometheus UI bằng basic auth
```
cat <<EOF > /tmp/prometheus/web.yml
basic_auth_users:
  admin: '$2a$12$noyKAVjH4kV9JUwRVbki7.4PJxz2rhJ1kZrac8uaJ75Y0cm3lYz46'
EOF
```
https://prometheus.io/docs/guides/basic-auth/#securing-prometheus-api-and-ui-endpoints-using-basic-auth

- Tạo Systemd Service cho Prometheus
```
cat << EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP \$MAINPID
ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/var/lib/prometheus \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries \
--web.listen-address=0.0.0.0:9090 \
--web.external-url= \
--web.config.file=/etc/prometheus/web.yml # require basic authentication for access metric url

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```


```
systemctl daemon-reload
systemctl enable prometheus.service
systemctl start prometheus.service
systemctl status prometheus.service
```

# Cài đặt Grafana

```
mkdir -p /tmp/grafana && cd /tmp/grafana
wget -q -O gpg.key https://rpm.grafana.com/gpg.key
rpm --import gpg.key

cat <<EOF > /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt

yum install grafana
```

- Start Grafana
```
systemctl start grafana-server
systemctl status grafana-server
```

# cài đặt Alert Manager

```
mkdir -p /tmp/alertmanager/ && cd /tmp/alertmanager/
wget https://github.com/prometheus/alertmanager/releases/download/v0.28.0-rc.0/alertmanager-0.28.0-rc.0.linux-amd64.tar.gz
tar -zxvf alertmanager-0.28.0-rc.0.linux-amd64.tar.gz
mv alertmanager-0.28.0-rc.0.linux-amd64/alertmanager /usr/local/bin
```


- Tạo Systemd Service cho Alert Manager
```
cat << EOF > /etc/systemd/system/alertmanager.service
[Unit]
Description=AlertManager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
--config.file=/etc/alertmanager/alertmanager.yml
[Install]
WantedBy=multi-user.target
EOF
```

- Start Alert Manager
```
systemctl daemon-reload
systemctl enable alertmanager
systemctl start alertmanager
systemctl status alertmanager
```





- Tạo bot telegram để nhận alert
![image](https://github.com/user-attachments/assets/e89ed675-dfe9-44da-84f9-b16643f2a9e3)

![image](https://github.com/user-attachments/assets/54d86652-5d9a-44d6-a18e-38fec1ee90f1)

Tạo 1 groupchat và add con bot vừa tạo vào groupchat đó

- Truy cập https://api.telegram.org/bot<telegram_token>/getUpdates
![image](https://github.com/user-attachments/assets/1f723217-ed93-40b5-9b15-ba541b5794c5)

- Quay lại Telegram và input `/start` với con bot

- Refresh lại browser để hiển thị chat ID của con bot và lưu lại chat ID của nó
![image](https://github.com/user-attachments/assets/1b738be1-26a0-41cc-90f9-52da22d48ccc)


```
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 1m # interval each time alert manager push notification
  repeat_interval: 1h
  receiver: 'telegram'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']

receivers:
- name: 'telegram'
  telegram_configs:
  - bot_token: '7544633848:AAEL7m3KikeQchx_0XxXIi_62us5mgdIlVQ'
    api_url: https://api.telegram.org
    chat_id: -4523984414
    message: '{{ template "telegram.yucca.message" . }}'

templates:
- '/etc/alertmanager/templates/telegram.tmpl'
```

telegram message template
```
{{ define "__yucca_text_alert_list" }}{{ range . }}
---
🪪 <b>{{ .Labels.alertname }}</b
{{- if .Annotations.summary }}
📝 {{ .Annotations.summary }}{{ end }}
{{- if .Annotations.description }}
📖 {{ .Annotations.description }}{{ end }}
🏷  Labels:
{{ range .Labels.SortedPairs }}  <i>{{ .Name }}</i>: <code>{{ .Value }}</code>
{{ end }}{{ end }}
🛠  <a href="https://yucca.app/">Grafana</a> 💊 <a href="https://yucca.app/">Alertmanager</a> 💊 <a href="https://yucca.app/">Prometheus</a> 🛠
{{ end }}

{{ define "telegram.yucca.message" }}
{{ if gt (len .Alerts.Firing) 0 }}
🔥 Alerts Firing 🔥
{{ template "__yucca_text_alert_list" .Alerts.Firing }}
{{ end }}
{{ if gt (len .Alerts.Resolved) 0 }}
✅ Alerts Resolved ✅
{{ template "__yucca_text_alert_list" .Alerts.Resolved }}
{{ end }}
{{ end }}
```
