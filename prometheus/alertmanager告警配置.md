## alertmanager告警配置

### 默认配置

```
global:
  resolve_timeout: 5m   ##全局配置，设置解析超时时间

route:
  group_by: ['alertname']  ##alertmanager中的分组，选哪个标签作为分组的依据
  group_wait: 10s          ##分组等待时间，拿到第一条告警后等待10s，如果有其他的一起发送出去
  group_interval: 10s    ##各个分组之前发搜告警的间隔时间
  repeat_interval: 1h    ##重复告警时间，默认1小时
  receiver: 'web.hook'   ##接收者

##配置告警接受者
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'

##配置告警收敛
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']

```

#### 邮件接收配置

```
global:
  resolve_timeout: 5m
  smtp_smarthost: smtp.163.com:25
  smtp_from: chenzongzheng163@163.com
  smtp_auth_username: chenzongzheng163@163.com
  smtp_auth_identity: chenzongzheng163@163.com
  smtp_auth_password: AYQAKURVNFUVHDTA
  smtp_hello: 163.com

route:
  receiver: 'email'
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 1h
  #alertmanager中的分组，选哪个标签作为分组的依据
  group_by: ['alertname']
  routes:
  - receiver: 'email'
    match_re:
      alertname: CPUThrottlingHigh|线上k8s集群Pod启动状态异常
    

receivers:
- name: email
  email_configs:
  - to: chenzongzheng163@163.com
    from: chenzongzheng163@163.com
    send_resolved: true
    headers:
      From: k8s-prometheus@163.com
      Subject: 'K8S集群告警'

inhibit_rules:
  - source_match:          
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance']
```

```
# cat alertmanager-email.yaml| base64 -w0
Z2xvYmFsOgogIHJlc29sdmVfdGltZW91dDogNW0KICAjIOmCrueusemFjee9rgogIHNtdHBfc21hcnRob3N0OiBzbXRwLjE2My5jb206MjUKICBzbXRwX2Zyb206IGNoZW56b25nemhlbmcxNjNAMTYzLmNvbQogIHNtdHBfYXV0aF91c2VybmFtZTogY2hlbnpvbmd6aGVuZzE2M0AxNjMuY29tCiAgc210cF9hdXRoX2lkZW50aXR5OiBjaGVuem9uZ3poZW5nMTYzQDE2My5jb20KICBzbXRwX2F1dGhfcGFzc3dvcmQ6IEFZUUFLVVJWTkZVVkhEVEEKICByZWNlaXZlcjogZW1haWwKICBncm91cF93YWl0OiAzMHMKICBncm91cF9pbnRlcnZhbDogMW0KICByZXBlYXRfaW50ZXJ2YWw6IDFoCnJvdXRlOgogICNhbGVydG1hbmFnZXLkuK3nmoTliIbnu4TvvIzpgInlk6rkuKrmoIfnrb7kvZzkuLrliIbnu4TnmoTkvp3mja4KCXJlY2VpdmVyOiAnZGVmYXVsdC1yZWNlaXZlcicgCiAgZ3JvdXBfYnk6IFsnYWxlcnRuYW1lJ10KICByb3V0ZXM6CiAgLSBtYXRjaDoKICAgICAgYWxlcnRuYW1lOiBDUFVUaHJvdHRsaW5nSGlnaHznur/kuIprOHPpm4bnvqRQb2TlkK/liqjnirbmgIHlvILluLgKICAgIHJlY2VpdmVyOiAnZW1haWwnCgpyZWNlaXZlcnM6Ci0gbmFtZTogZW1haWwKICBlbWFpbF9jb25maWdzOgogIC0gdG86IGNoZW56b25nemhlbmcxNjNAMTYzLmNvbQogICAgZnJvbTogY2hlbnpvbmd6aGVuZzE2M0AxNjMuY29tCiAgICBoZWFkZXJzOgogICAgICBGcm9tOiBrOHMtcHJvbWV0aGV1c0AxNjMuY29tCiAgICAgIFN1YmplY3Q6ICdLOFPpm4bnvqTlkYroraYnCg==
```

```
# cat alertmanager-secret.yaml 
apiVersion: v1
data:
  alertmanager.yaml: Z2xvYmFsOgogIHJlc29sdmVfdGltZW91dDogNW0KICAjIOmCrueusemFjee9rgogIHNtdHBfc21hcnRob3N0OiBzbXRwLjE2My5jb206MjUKICBzbXRwX2Zyb206IGNoZW56b25nemhlbmcxNjNAMTYzLmNvbQogIHNtdHBfYXV0aF91c2VybmFtZTogY2hlbnpvbmd6aGVuZzE2M0AxNjMuY29tCiAgc210cF9hdXRoX2lkZW50aXR5OiBjaGVuem9uZ3poZW5nMTYzQDE2My5jb20KICBzbXRwX2F1dGhfcGFzc3dvcmQ6IEFZUUFLVVJWTkZVVkhEVEEKICByZWNlaXZlcjogZW1haWwKICBncm91cF93YWl0OiAzMHMKICBncm91cF9pbnRlcnZhbDogMW0KICByZXBlYXRfaW50ZXJ2YWw6IDFoCnJvdXRlOgogICNhbGVydG1hbmFnZXLkuK3nmoTliIbnu4TvvIzpgInlk6rkuKrmoIfnrb7kvZzkuLrliIbnu4TnmoTkvp3mja4KCXJlY2VpdmVyOiAnZGVmYXVsdC1yZWNlaXZlcicgCiAgZ3JvdXBfYnk6IFsnYWxlcnRuYW1lJ10KICByb3V0ZXM6CiAgLSBtYXRjaDoKICAgICAgYWxlcnRuYW1lOiBDUFVUaHJvdHRsaW5nSGlnaHznur/kuIprOHPpm4bnvqRQb2TlkK/liqjnirbmgIHlvILluLgKICAgIHJlY2VpdmVyOiAnZW1haWwnCgpyZWNlaXZlcnM6Ci0gbmFtZTogZW1haWwKICBlbWFpbF9jb25maWdzOgogIC0gdG86IGNoZW56b25nemhlbmcxNjNAMTYzLmNvbQogICAgZnJvbTogY2hlbnpvbmd6aGVuZzE2M0AxNjMuY29tCiAgICBoZWFkZXJzOgogICAgICBGcm9tOiBrOHMtcHJvbWV0aGV1c0AxNjMuY29tCiAgICAgIFN1YmplY3Q6ICdLOFPpm4bnvqTlkYroraYnCg==
kind: Secret
metadata:
  name: alertmanager-main
  namespace: monitoring
type: Opaque
# kubectl apply -f alertmanager-secret.yaml
secret/alertmanager-main configured
```

或者命令行配置secret

```
kubectl -n monitoring create secret generic alertmanager-main --from-file=alertmanager-email.yaml
secret/alertmanager-inst created
```

Alertmanager将以下JSON格式的HTTP POST请求发送到配置的端点

```
{
  "version": "4",
  "groupKey": <string>,              // key identifying the group of alerts (e.g. to deduplicate)
  "truncatedAlerts": <int>,          // how many alerts have been truncated due to "max_alerts"
  "status": "<resolved|firing>",
  "receiver": <string>,
  "groupLabels": <object>,
  "commonLabels": <object>,
  "commonAnnotations": <object>,
  "externalURL": <string>,           // backlink to the Alertmanager.
  "alerts": [
    {
      "status": "<resolved|firing>",
      "labels": <object>,
      "annotations": <object>,
      "startsAt": "<rfc3339>",
      "endsAt": "<rfc3339>",
      "generatorURL": <string>       // identifies the entity that caused the alert
    },
    ...
  ]

```

Email.tmpl

```
{{ define "email.html" }}

<style type="text/css">
table.gridtable {
  font-family: verdana,arial,sans-serif;
  font-size:11px;
  color:#333333;
  border-width: 1px;
  border-color: #666666;
  border-collapse: collapse;
}
table.gridtable th {
  border-width: 1px;
  padding: 8px;
  border-style: solid;
  border-color: #666666;
  background-color: #dedede;
}
table.gridtable td {
  border-width: 1px;
  padding: 8px;
  border-style: solid;
  border-color: #666666;
  background-color: #ffffff;
}
</style>

<table class="gridtable">
  <tr>
    <th>报警项</th>
    <th>实例</th>
    <th>当前值</th>
    <th>告警级别</th>
    <th>开始时间</th>
    <th>告警摘要</th>
  </tr>
  {{ range $i, $alert := .Alerts }}
    <tr>
      <td>{{ index $alert.Labels "alertname" }}</td>
      <td>{{ index $alert.Labels "container" }}</td>
      <td>{{ index $alert.Annotations "value" }}</td>
      <td>{{ index $alert.Labels "severity" }}</td>
      <td>{{ $alert.StartsAt.Format "2006-01-02 15:04:05 MST" }}</td>
      <td>{{ index $alert.Annotations "message" }}</td>
    </tr>
  {{ end }}
</table>
{{ end }}
```

