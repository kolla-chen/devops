## 虚拟机inband带宽测试

- 登入地市节点的 in-band-net1

  ```
  [root@szx-in-band-net1 ~]# 
  ```

- 开启临时测试端口5201-5204端口

  ```
  [root@szx-in-band-net1 ~]# firewall-cmd --add-port=5201-5204/tcp
  ```

- 安装iperf3并开启四个server

  ```
  yum -y install iperf3 
  
  iperf3 -s -p 5201 --logfile /tmp/5201.log -D
  iperf3 -s -p 5202 --logfile /tmp/5202.log -D
  iperf3 -s -p 5203 --logfile /tmp/5203.log -D
  iperf3 -s -p 5204 --logfile /tmp/5204.log -D
  ```
  
- 从lvs和cache抽出4台做为客户端 同时对 in-band-net1 进行压测

  ```
  [axe@szx-lvs1 ~]$ iperf3 -c 10.128.1.240  -Z -R -b 50G -P 30 -O 5 -t 300 -p 5201
  [axe@szx-lvs2 ~]$ iperf3 -c 10.128.1.240  -Z -R -b 50G -P 30 -O 5 -t 300 -p 5202 
  [axe@szx-cache1 ~]$ iperf3 -c 10.128.1.240  -Z -R -b 50G -P 30 -O 5 -t 300 -p 5203
  [axe@szx-cache2 ~]$ iperf3 -c 10.128.1.240  -Z -R -b 50G -P 30 -O 5 -t 300 -p 5204
  ```

- 查看测试结果

  ```
  [root@evm-kuaishou-instance ~]# grep sender /tmp/520*.log |grep -i sum
  /tmp/5201.log:[SUM]   0.00-300.01 sec   188 GBytes  5.37 Gbits/sec   61             sender
  /tmp/5202.log:[SUM]   0.00-300.00 sec  94.1 GBytes  2.69 Gbits/sec   34             sender
  /tmp/5203.log:[SUM]   0.00-300.01 sec   183 GBytes  5.25 Gbits/sec   96             sender
  /tmp/5204.log:[SUM]   0.00-300.01 sec   262 GBytes  7.49 Gbits/sec  11233             sender
  ```

  

- 记录表格

  | Client     | Server       | Bandwidth ( Gbits/sec) | Sum Bandwidth ( Gbits/sec) |
  | ---------- | ------------ | ---------------------- | -------------------------- |
  | 10.128.1.1 | 10.128.1.240 | 3.36                   | 13.18                      |
  | 10.128.1.2 | 10.128.1.240 | 3.24                   |                            |
  | 10.128.1.3 | 10.128.1.240 | 3.23                   |                            |
  | 10.128.1.4 | 10.128.1.240 | 3.35                   |                            |

-  in-band-net1 防火墙恢复配置

  ```
  [root@szx-in-band-net1 ~]# firewall-cmd --reload
  ```

  