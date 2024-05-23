monitoring http, ping, node, mysql and etc with docker compose using prometheus and grafana

------------
- install mysql exporter for nodes:

MYSQL_EXPORTER_VERSION=$(curl -sL https://api.github.com/repos/prometheus/mysqld_exporter/releases/latest | grep "tag_name"   | sed -E 's/.*"([^"]+)".*/\1/'|sed 's/v//')

tar xvf mysqld_exporter*.tar.gz
cd mysqld_exporter*.*-amd64
cp mysqld_exporter /usr/bin/

nano /etc/mysqld_exporter.cnf
[client]
user=exporter
password=StrongPassssss

mysql -u root -p
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'StrongPassssss' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;

nano /lib/systemd/system/mysqld_exporter.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/bin/mysqld_exporter \
--config.my-cnf /etc/mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9104

[Install]
WantedBy=multi-user.target

chmod 664 /lib/systemd/system/mysqld_exporter.service
systemctl daemon-reload
systemctl start mysql_exporter
systemctl status mysql_exporter
systemctl enable mysql_exporter

firewall-cmd --permanent --zone=public --add-port=9104/tcp
firewall-cmd --reload

- Verify mysql exporter is Running:
curl http://localhost:9104/metrics

------------
- install node exporter full resources linux host:

sudo groupadd -f node_exporter
sudo useradd -g node_exporter --no-create-home --shell /bin/false node_exporter
sudo mkdir /etc/node_exporter
sudo chown node_exporter:node_exporter /etc/node_exporter

wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.*-amd64.tar.gz
cd node_exporter-*.*-amd64
cp node_exporter /usr/bin/
sudo chown node_exporter:node_exporter /usr/bin/node_exporter

SERVICE_TEXT="[Unit]
 Description=Node Exporter
 Documentation=https://prometheus.io/docs/guides/node-exporter/
 Wants=network-online.target
 After=network-online.target

[Service]
 User=node_exporter
 Group=node_exporter
 Type=simple
 Restart=on-failure
 ExecStart=/usr/bin/node_exporter --web.listen-address=:9100

[Install]
 WantedBy=multi-user.target"

echo "$SERVICE_TEXT" | sudo tee /lib/systemd/system/node_exporter.service > /dev/null

chmod 664 /lib/systemd/system/node_exporter.service

systemctl daemon-reload
systemctl start node_exporter
systemctl status node_exporter
systemctl enable node_exporter

firewall-cmd --permanent --zone=public --add-port=9100/tcp
firewall-cmd --reload

- Verify Node Exporter is Running:
http://<node_exporter-ip>:9100/metrics

------------
- install blackbox exporter:

useradd --no-create-home --shell /bin/false blackbox_exporter

cd ~
curl -LO https://github.com/prometheus/blackbox_exporter/releases/download/v0.12.0/blackbox_exporter-0.12.0.linux-amd64.tar.gz
check checksums:
sha256sum blackbox_exporter-0.12.0.linux-amd64.tar.gz
Output
c5d8ba7d91101524fa7c3f5e17256d467d44d5e1d243e251fd795e0ab4a83605  blackbox_exporter-0.12.0.linux-amd64.tar.gz

tar xvf blackbox_exporter-0.12.0.linux-amd64.tar.gz
mv ./blackbox_exporter-0.12.0.linux-amd64/blackbox_exporter /usr/local/bin
chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
rm -rf ~/blackbox_exporter-0.12.0.linux-amd64.tar.gz ~/blackbox_exporter-0.12.0.linux-amd64

mkdir /etc/blackbox_exporter
chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
nano /etc/blackbox_exporter/blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:      
      valid_status_codes: []
      method: GET

chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter/blackbox.yml
nano /etc/systemd/system/blackbox_exporter.service
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl start blackbox_exporter
systemctl status blackbox_exporter
systemctl enable blackbox_exporter

firewall-cmd --permanent --zone=public --add-port=9115/tcp
firewall-cmd --reload

- Verify blackbox Exporter is Running:
http://<node_exporter-ip>:9115/metrics

------------
- install supervisor exporter:

nano /etc/supervisor/supervisord.conf
[inet_http_server]
port = 127.0.0.1:9001

supervisorctl reread
supervisorctl update

curl http://127.0.0.1:9001/RPC2
git clone https://github.com/salimd/supervisord_exporter.git
go build

mkdir -p /etc/supervisor/logs/

nano /etc/supervisor/conf.d/supervisor-metrics.conf
[program:supervisor-metrics]
process_name=%(program_name)s_%(process_num)02d
command=/root/supervisor/supervisord_exporter/supervisord_exporter -supervisord-url="http://127.0.0.1:9001/RPC2" -web.listen-address=":9200" -web.telemetry-path="/metrics"
autostart=true
autorestart=true
user=root
numprocs=1
redirect_stderr=true
stdout_logfile=/etc/supervisor/logs/supervisor-metrics.log
stopwaitsecs=3600
startsecs=0

supervisorctl reread
supervisorctl update

- Verify Node Exporter is Running:
http://<node_exporter-ip>:9200
http://<node_exporter-ip>:9200/metrics

Source: https://github.com/salimd/supervisord_exporter
