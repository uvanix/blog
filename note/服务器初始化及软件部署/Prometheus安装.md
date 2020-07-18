## Prometheus安装
```
# 下载
cd /opt && wget https://github.com/prometheus/prometheus/releases/download/v2.10.0/prometheus-2.10.0.linux-amd64.tar.gz
 
# 解压
tar -zxvf prometheus-2.10.0.linux-amd64.tar.gz && mv prometheus-2.10.0.linux-amd64 prometheus
 
# 验证安装
cd /opt/prometheus && ./prometheus --version
 
# 创建用户并授权
# 这里单独创建一个专门用于运行prometheus的用户，不用root运行程序是一种好习惯。主目录为/opt/prometheus/data，用作prometheus的数据目录
mkdir -p /opt/prometheus/data
groupadd prometheus
useradd prometheus -g prometheus -M -s /sbin/nologin
chown -R prometheus.prometheus /opt/prometheus
 
# 创建 prometheus 系统服务启动文件 /usr/lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
 
[Service]
User=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/data
ExecReload=/bin/kill -HUP $MAINPID
[Install]
WantedBy=multi-user.target
 
# 修改 prometheus 配置文件 /opt/prometheus/prometheus.yml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
 
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["localhost:9093"]
 
rule_files:
  #- "alert.rules"
   
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval:     5s
    static_configs:
      - targets: ['localhost:9090']
 
  - job_name: 'node'
    scrape_interval:     10s
    static_configs:
      - targets: ['localhost:9100']
 
 
# 启动服务
systemctl daemon-reload
systemctl start prometheus.service
systemctl enable prometheus.service
systemctl status prometheus.service
Prometheus 服务支持热加载配置：systemctl reload prometheus.service
Prometheus 服务启动完成后，可以通过http://localhost:9090访问 Prometheus 的 UI 界面。
```

## 安装配置 node_exporter

为监控服务器 CPU , 内存 , 磁盘 , I/O 等信息，需要在被监控机器上安装 node_exporter 服务。
```
# 下载
cd /opt && wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
 
# 解压
tar -zxvf node_exporter-0.18.1.linux-amd64.tar.gz && mv node_exporter-0.18.1.linux-amd64 node_exporter
groupadd prometheus
useradd prometheus -g prometheus -M -s /sbin/nologin
chown -R prometheus.prometheus /opt/node_exporter
 
# 创建 node_exporter 系统服务启动文件 /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
 
[Service]
User=prometheus
ExecStart=/opt/node_exporter/node_exporter
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
 
# 启动 node_exporter 服务
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
服务启动后可以用 http://localhost:9100/metrics 测试 node_exporter 是否获取到节点的监控指标
如果可以正常获取到节点的指标后，我们可以将 node_exporter 整合到 prometheus 中
 
修改 prometheus 的配置文件/opt/prometheus/prometheus.yml，增加如下内容：
 
scrape_configs:
...
- job_name: 'node'
    scrape_interval:     10s
    static_configs:
      - targets: ['localhost:9100']
 
重启 Prometheus 服务：
systemctl reload prometheus.service
之后就可以通过 Prometheus 服务获取该主机的相关资源了
```