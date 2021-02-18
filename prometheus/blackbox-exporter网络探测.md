# blackbox-exporter网络探测

### 黑盒监控

相较于白盒监控最大的不同在于黑盒监控是以故障为导向当故障发生时，黑盒监控能快速发现故障，而白盒监控则侧重于主动发现或者预测潜在的问题。一个完善的监控目标是要能够从白盒的角度发现潜在问题，能够在黑盒的角度快速发现已经发生的问题。

Blackbox Exporter是Prometheus社区提供的官方黑盒监控解决方案，可以提供 http、dns、tcp、icmp 、ssl的方式对网络进行探测。

[github官方](https://www.liuyalei.top/wp-content/themes/begin/inc/go.php?url=https://github.com/prometheus/blackbox_exporter)

### 部署blackbox_exporter

[具体配置参考](https://github.com/prometheus/blackbox_exporter/blob/master/example.yml)

```
[root@dev-node-01 black_exporter]# cat blackbox-exporter.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: blackbox-exporter
  namespace: monitoring
data:
  config.yml: |
    modules:
      http_2xx:
        prober: http
        http:
          method: GET
          preferred_ip_protocol: "ip4"
          valid_http_versions: ["HTTP/1.1", "HTTP/2"]
          valid_status_codes: [200,301,302]          
      http_post_2xx:
        prober: http
        http:
          method: POST
          preferred_ip_protocol: "ip4"
      tcp_connect:
        prober: tcp
      icmp:
        prober: icmp
        timeout: 3s
        icmp:
          preferred_ip_protocol: "ip4"
      dns_tcp:
        prober: dns
        timeout: 5s
        dns:
          transport_protocol: "tcp"
          preferred_ip_protocol: "ip4"
          query_name: "kubernetes.default.svc.cluster.local"
          query_type: "A"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: blackbox-exporter
  name: blackbox-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      name: blackbox-exporter
  template:
    metadata:
      labels:
        name: blackbox-exporter
    spec:
      containers:
      - image: prom/blackbox-exporter:v0.18.0
        name: blackbox-exporter
        ports:
        - containerPort: 9115
        volumeMounts:
        - name: config
          mountPath: /etc/blackbox_exporter
        args:
        - --config.file=/etc/blackbox_exporter/config.yml
        - --log.level=info
      volumes:
      - name: config
        configMap:
          name: blackbox-exporter
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: blackbox-exporter
  name: blackbox-exporter
  namespace: monitoring
spec:
  selector:
    name: blackbox-exporter
  ports:
  - name: http-metrics
    port: 9115
    targetPort: 9115
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: blackbox-exporter-ingress
  namespace: monitoring
spec:
  rules:
  - host: blackbox-exporter.kolla.top
    http:
      paths:
      - backend:
          serviceName: blackbox-exporter
          servicePort: 9115
```

```
kubectl apply -f blackbox-exporter.yaml
kubectl get svc -n monitoring
kubectl get ingress -n monitoring
kubectl get deploy -n monitoring
```

### 自定义配置job

[github官方](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/additional-scrape-config.md)

```
[root@dev-node-01 black_exporter]# cat prometheus-additional.yaml 
##检查http网站存活
- job_name: "blackbox-external-website"
  scrape_interval: 30s
  scrape_timeout: 15s
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
  - targets:
    - https://www.example.com # 要检查的网址
    - https://test.example.com
    - https://www.baidu.com
    - http://www.sina.com.cn
    - http://www.liuyalei.top
      labels:
        group: 'website'
  relabel_configs:
  - source_labels: [__address__]
    target_label: __param_target
  - source_labels: [__param_target]
    target_label: instance
  - source_labels:
    - __param_module
    target_label: module
    regex: (.*)
    replacement: $1
    action: replace
  - target_label: __address__
    replacement: blackbox-exporter:9115
    
##检查主机存活
- job_name: 'blackbox-node-status'
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets: ['180.114.8.1','192.168.8.101','180.114.8.2']
      labels:
        group: 'ecf'
  relabel_configs:
  - source_labels: [__address__]
    target_label: __param_target
  - source_labels: [__param_target]
    target_label: instance
  - source_labels:
    - __param_module
    target_label: module
    regex: (.*)
    replacement: $1
    action: replace
  - target_label: __address__
    replacement: blackbox-exporter:9115  
 
##检查主机端口存活
- job_name: 'balckbox-port-status'
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
    - targets: ['180.114.8.1:9100','172.16.1.80:80','172.16.1.82:22']
      labels:
        group: 'tcp'
  relabel_configs:
  - source_labels: [__address__]
    target_label: __param_target
  - source_labels: [__param_target]
    target_label: instance
  - source_labels:
    - __param_module
    target_label: module
    regex: (.*)
    replacement: $1
    action: replace
  - target_label: __address__
    replacement: blackbox-exporter:9115   #endpoint改回 blackbox 的service+port不然会导致探测失败 
```

```
kubectl create secret generic additional-scrape-configs --from-file=prometheus-additional.yaml -n monitoring --dry-run -oyaml > additional-scrape-configs.yaml

kubectl apply -f additional-scrape-configs.yaml
```

### 修改operator`prometheus.yaml` CRD.配置

```
[root@dev-node-01 black_exporter]# more ../manifests/prometheus-prometheus.yaml 
apiVersion: monitoring.coreos.com/v1
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  labels:
    prometheus: prometheus
spec:
  replicas: 2
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: prometheus-additional.yaml
...
```

```
 kubectl apply -f ../manifests/prometheus-prometheus.yaml
```

### 另外一种方法【serviceMonitor多主机探测写法不便】

```
[root@dev-node-01 black_exporter]# cat black-exporter-serviceMonitor.yaml 
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox-exporter
  namespace: monitoring
spec:
  endpoints:
  #检查网站存活curl
  - interval: 30s  #多长时间抓取一次
    params:
      module:
        - http_2xx   #抓取模块
      target:
        - https://blog.csdn.net
    path: "/probe"
    port: http-metrics   #定义endpoint名称
    scheme: http    #抓取的方法
    scrapeTimeout: 30s   #抓取超时时间
    relabelings:
      - sourceLabels:
          - __param_target
        targetLabel: target
      - sourceLabels:
          - __param_target
        targetLabel: instance
      - action: replace
        regex: (.*)
        replacement: $1
        sourceLabels:
        - __param_module
        targetLabel: module        
  #检查主机存活ping
  - interval: 30s
    params:
      module:
        - icmp
      target:
        - 192.168.8.11 # 多主机这些写不会生效，原因不明
        - 192.168.254.1
        - 192.168.56.1
    path: "/probe"
    port: http-metrics
    scrapeTimeout: 30s
    relabelings:
      - action: replace
        regex: (.*)
        replacement: $1
        sourceLabels:
        - __param_module
        targetLabel: module
      - action: replace
        regex: (.*)
        replacement: $1
        sourceLabels:
        - __param_target
        targetLabel: instance
  #检查端口存活telnet
  - interval: 30s
    params:
      module:
        - tcp_connect
      target:
        - 192.168.8.11:22
    path: "/probe"
    port: http-metrics
    scrapeTimeout: 30s
    relabelings:
      - sourceLabels:
          - __param_target
        targetLabel: target
      - sourceLabels:
          - __param_target
        targetLabel: instance
       - action: replace
        regex: (.*)
        replacement: $1
        sourceLabels:
        - __param_module
        targetLabel: module
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      name: blackbox-exporter    
      #选择blacebox exporter容器service的标签！！！svc记得打标签，否则匹配不到
```

### 告警规则

```

# kube-ovn Controller
  - name: kubernetes-blackbox
    rules:
    - alert: blackbox 探测失败
      expr: probe_success == 0
      for: 15m
      labels:
        severity: "critical"
        obj: "{{ $labels.name }}"
        cluster: "fj-fuzhou-dev"
      annotations:
        value: "{{ $value }}"
        summary: "{{ $labels.job }} {{ $labels.module }}: {{ $labels.instance }}"
        description: "探测任务{{ $labels.module }} 目标 {{ $labels.instance }} Down"
```

consul 自动注册

```
cat >node_exporter-1.json<<EOF
{
  "ID": "node-exporter",
  "Name": "node-exporter",
  "Tags": [
    "master1"
  ],
  "Address": "172.16.1.41",
  "Port": 9100,
  "Meta": {
    "app": "elasticsearch",
    "team": "bigdata",
    "project": "log-analyze"
  },
  "EnableTagOverride": false,
  "Check": {
    "HTTP": "http://172.16.1.41:9100/metrics",
    "Interval": "10s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}

EOF
curl -X PUT -d @node_exporter-1.json http://172.16.1.83:32471/v1/agent/service/register
```

