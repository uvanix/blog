## 下载grafana安装启动
```
cd /opt && wget https://dl.grafana.com/oss/release/grafana-6.2.2-1.x86_64.rpm
sudo yum -y localinstall grafana-6.2.2-1.x86_64.rpm
service grafana-server start
```


服务启动后 grafana 默认监听在 3000 端口 ，可以通过 http://127.0.0.1:3000 访问 grafana 的 ui 界面，默认登录账号密码为 admin/admin ，第一次登录需要我们重置密码。修改为admin

当 grafana 的页面可以正常访问后，我们就可以添加数据源了，具体操作流程如下：

“Configration”—“Data Sources” 然后根据需要进行配置，需要注意的是 prometheus 的地址需要根据实际情况做修改。

grafana 的数据源配置完成后，可以导入一个 dashboard 模板文件，建议节点模板使用 [node_exporter 展示面板模板](https://grafana.com/grafana/dashboards/8919)
