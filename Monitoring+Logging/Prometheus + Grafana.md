# Cài đặt Node Exporter
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
ExecStart=/usr/local/bin/node_exporter --collector.systemd

[Install]
WantedBy=multi-user.target
EOF
```

- Start node_exporter
```
systemctl daemon-reload
systemctl enable node-exporter.service
systemctl start node-exporter.service
systemctl status node-exporter.service
```


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
--web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```


```
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
```
