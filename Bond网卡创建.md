## 1.创建Bond0网卡

### 1.1子网卡作为slave

```shell
~]# cat /etc/sysconfig/network-scripts/ifcfg-ens2f0
#设备名称
DEVICE=ens2f0
BOOTPROTO=static
ONBOOT=yes
USERCTL=no
MASTER=Bond0
SLAVE=yes
#网卡物理地址
HWADDR=48:dc:2d:05:f8:8b
#网卡名称（同设备名称）
NAME=ens2f0
```

### 1.2逻辑网卡Bond0作为master

```shell
 ~]# cat /etc/sysconfig/network-scripts/ifcfg-Bond0
DEVICE=Bond0
TYPE=Ethernet
ONBOOT=yes
IPV6INIT=no
BONDING_MASTER=yes
BONDING_OPTS="mode=802.3ad miimon=100 xmit_hash_policy=layer3+4"
```

### 1.3创建带vlan的子网卡

```shell
~]# cat /etc/sysconfig/network-scripts/ifcfg-Bond0.10 
DEVICE=Bond0.10
TYPE=Vlan
PHYSDEV=Bond0
ONBOOT=yes
BOOTPROTO=static
REORDER_HDR=yes
#ip 地址
IPADDR=180.114.8.1
#掩码
PREFIX=25
#网关&内网没网关就不配
GATEWAY=180.114.8.126
IPV6INIT=no
BONDING_MASTER=yes
BONDING_OPTS="mode=802.3ad miimon=100 xmit_hash_policy=layer3+4"
VLAN=yes
VLANID=10
```

### 1.4重启网络服务

```shell
~]# systemctl stop NetworkManager
~]# systemctl disable NetworkManager
~]# systemctl restart network 
~]# systemctl enable network 
```

