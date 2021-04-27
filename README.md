# supervisord_prometheus_monitor
收集supervisor的进程状态信息，并将信息暴露给Prometheus

#安装依赖模块
*python3环境
*pip install prometheus-client -i https://pypi.tuna.tsinghua.edu.cn/simple/

#supervisord配置  
```
cat /etc/supervisord.d/prometheus_exporter.ini 

[program:supervisor_exporter]
process_name=%(program_name)s
command=/usr/bin/python3 /root/scripts/supervisor_exporter.py
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/supervisor/supervisor_exporter.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=3
buffer_size=10
```

#配置job  
```
  - job_name: 'supervistor-exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.101.100:8000']
```
或者通过consul自动发现监控

```
cat prome.json 
{
  "ID": "192.168.101.100_supervisord",
  "Name": "192.168.101.100_supervisord",
  "Tags": [
    "192.168.101.100_supervisord"
  ],
  "Address": "192.168.101.100",
  "Port": 8000,
  "EnableTagOverride": false,
  "Check": {
    "HTTP": "http://192.168.101.100:8000/metrics",
    "Interval": "10s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}

```
向consul注册监控主机：
```
curl --request PUT --data @prome.json http://192.168.101.200:8500/v1/agent/service/register?replace-existing-checks=1
```
配置job正则匹配
```
  - job_name: 'supervisord'
    scrape_interval: 5s
    consul_sd_configs:
      - server: '192.168.101.200:8500'
        services: []
    relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*_supervisord.*
      action: keep
```

配置报警规则
```
groups:
- name: supervisord
  rules:
  - alert: supervisord monitor
    expr: supervisord_up == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Job {{ $labels.group }} down"
      description: "{{ $labels.group }} of job {{ $labels.job }} has been down for more than 2 minutes."
```

#其他资料
*https://github.com/prometheus/client_python