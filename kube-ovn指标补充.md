## kube-ovn监控指标分析

### 采集target Down

```
count by(job, namespace, service, instance) (up == 0) 
```

### controller 增删改操作频率
```
workqueue_adds_total
workqueue_retries_total
```

### controller 工作队列延迟
```
workqueue_queue_duration_seconds_sum
workqueue_queue_duration_seconds_count
```

### controller api4xx,5xx错误率
```
rest_client_requests_total
```

### cni 动作4xx,5xx状态统计
```
cni_op_latency_seconds_count
```

### ovsovn pinger down
```
pinger_ovs_down
pinger_ovn_controller_down
```

### kube-ovn 内部dns健康

```
pinger_internal_dns_healthy
```

### kube-ovn pinger apiserver 平均延迟ms
```
pinger_apiserver_latency_ms_sum
pinger_apiserver_latency_ms_count
```

### ovn leader
```
kube_ovn_cluster_role
```

### ovn db size
```
kube_ovn_db_file_size_bytes
```

### ovn log size
```
kube_ovn_log_file_size_bytes
```

