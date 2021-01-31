### openipmi

```shell
#安装openipmi
apt-get install openipmi

#加载ipmi驱动模块
 modprobe ipmi_msghandler
 modprobe ipmi_devintf
 modprobe ipmi_si
 modprobe ipmi_poweroff
 modprobe ipmi_watchdog

#显示IPMI网络设置
ipmitool lan print

# 显示IPMI用户
ipmitool user list 1

#设置ID位2的用户密码位123456
ipmitool user set password 2 "calvin"

#设置IPMI通道1的地址
ipmitool lan set 1 ipaddr 192.168.254.168

#设置IPMI通道1的子网掩码
ipmitool lan set 1 netmask 255.255.255.0

#设置IPMI通道1的IP网关
ipmitool lan set 1 defgw ipaddr 192.168.254.1

# 关闭vlanid
ipmitool lan set 1 vlan id off


```



### 块设备的使用

创建RBD 存储池

```
ceph osd pool create <pool-name> 50 50 # 两个 50 指定的 pg 和 pgp 的数量
ceph osd pool application enable <pool-name> rbd
```

```shell
 ceph osd pool create rbd 128 128
 ceph osd pool application enable rbd rbd
 # 或者用rados
 rados rmpool rbd rbd  --yes-i-really-really-mean-it
 rados mkpool rbd 
 rbd pool init rbd
```

导出red 客户端密钥

```
ceph auth get-or-create client.rbd -o ./ceph.client.rbd.keyring 
```

把秘钥发送到客户端

```
 cp ceph.client.rbd.keyring /etc/ceph/
 scp ceph.client.rbd.keyring hostip:/etc/ceph/
```

查看权限信息，赋予权限

```
 ceph auth get client.rbd
 ceph auth caps  client.rbd  mon 'allow r' osd 'allow rwx pool=rbd' 
 ceph auth get client.rbd
 ceph osd pool ls --id rbd
 rbd ls rbd --id rbd
```

创建块设备，映射到系统

```
 rbd create --size 1G rbd/testimg  --id rbd
 rbd ls rbd --id rbd
 rbd map  rbd/testimg  --id rbd 

 lsblk /dev/rbd0 
 mkfs.xfs /dev/rbd0 
 blkid  /dev/rbd0
 mount /dev/rbd0  /mnt/

 rbd info rbd/testimg --id rbd
```

### 常用RBD 操作

#### 获取映像列表

要挂载块设备映像，先罗列出所有的映像。

```
rbd list
```

#### 映射块设备

用 `rbd` 把映像名映射为内核模块。必须指定映像名、存储池名、和用户名。若 RBD 内核模块尚未加载， `rbd` 命令会自动加载。

```
sudo rbd map {pool-name}/{image-name} --id {user-name}
```

例如：

```
sudo rbd map rbd/myimage --id admin
```

如果你启用了 [cephx](http://docs.ceph.org.cn/rados/configuration/auth-config-ref/) 认证，还必须提供密钥，可以用密钥环或密钥文件指定密钥。

```
sudo rbd map rbd/myimage --id admin --keyring /path/to/keyring
sudo rbd map rbd/myimage --id admin --keyfile /path/to/file
```

#### 查看已映射块设备[¶](http://docs.ceph.org.cn/rbd/rbd-ko/#id4)

可以用 `rbd` 命令的 `showmapped` 选项查看映射为内核模块的块设备映像。

```
rbd showmapped
```

#### 取消块设备映射

要取消块设备映射，用 `rbd` 命令、指定 `unmap` 选项和设备名（即为方便起见使用的同名块设备映像）。

```
sudo rbd unmap /dev/rbd/{poolname}/{imagename}
```

例如：

```
sudo rbd unmap /dev/rbd/rbd/foo
```

 

#### 创建块设备映像

要想把块设备加入某节点，你得先在 [*Ceph 存储集群*](http://docs.ceph.org.cn/glossary/#term-21)中创建一个映像，使用下列命令：

```
rbd create --size {megabytes} {pool-name}/{image-name}
```

例如，要在 `swimmingpool` 这个存储池中创建一个名为 `bar` 、大小为 1GB 的映像，执行：

```
rbd create --size 1024 swimmingpool/bar
```

如果创建映像时不指定存储池，它将使用默认的 `rbd` 存储池。例如，下面的命令将默认在 `rbd` 存储池中创建一个大小为 1GB 、名为 `foo` 的映像：

```
rbd create --size 1024 foo
```

指定此存储池前必须先创建它，详情见[存储池](http://docs.ceph.org.cn/rados/operations/pools)。

#### 罗列块设备映像

要列出 `rbd` 存储池中的块设备，可以用下列命令（即 `rbd` 是默认存储池名字）：

```
rbd ls
```

用下列命令罗列某个特定存储池中的块设备，用存储池的名字替换 `{poolname}` ：

```
rbd ls {poolname}
```

例如：

```
rbd ls swimmingpool
```

#### 检索映像信息

用下列命令检索某个特定映像的信息，用映像名字替换 `{image-name}` ：

```
rbd info {image-name}
```

例如：

```
rbd info foo
```

用下列命令检索某存储池内的映像的信息，用映像名字替换 `{image-name}` 、用存储池名字替换 `{pool-name}` ：

```
rbd info {pool-name}/{image-name}
```

例如：

```
rbd info swimmingpool/bar
```

#### 调整块设备映像大小

[*Ceph 块设备*](http://docs.ceph.org.cn/glossary/#term-38)映像是精简配置，只有在你开始写入数据时它们才会占用物理空间。然而，它们都有最大容量，就是你设置的 `--size` 选项。如果你想增加（或减小） Ceph 块设备映像的最大尺寸，执行下列命令：

```
rbd resize --size 2048 foo (to increase)
rbd resize --size 2048 foo --allow-shrink (to decrease)
```

#### 删除块设备映像

可用下列命令删除块设备，用映像名字替换 `{image-name}` ：

```
rbd rm {image-name}
```

例如：

```
rbd rm foo
```

用下列命令从某存储池中删除一个块设备，用要删除的映像名字替换 `{image-name}` 、用存储池名字替换 `{pool-name}` ：

```
rbd rm {pool-name}/{image-name}
```

例如：

```
rbd rm swimmingpool/bar
```

**调整RBD镜像大小**

在上面的映射完块设备格式化挂载后，使用resize命令调整RBD，然后用XFS在线调整特性扩容文件系统。

```
rbd resize volume rbd/rbd_test  --size 20 
xfs_growfs -d  /mnt/rbd_test
```



### iscsi-gw服务端

#### 部署iscsi-gw

部署网关可以独立节点也可以和集群融合。

初始化后，上传ceph-iscsi-mimic.tar.gz

```shell

yum install -y ceph-common librados2 librados2-devel librbd1 librbd1-devel python-rados python-rbd 

yum install -y cmake make gcc \
libnl3 libnl3-devel glib2 glib2-devel zlib zlib-devel kmod kmod-devel \
glusterfs-api glusterfs-api-devel python-setuptools python2-cryptography

yum install -y libkmod pyparsing python-kmod python-pyudev python-gobject \
python-urwid python-pyparsing python-netaddr python-netifaces \
python-crypto python-requests python-flask pyOpenSSL

cd ceph-iscsi
yum localinstall *.rpm

```



#### 配置iscsi

iscsi网关节点必须要有rbd的密钥文件+ceph.conf配置文件

```shell
# mon节点
scp /etc/ceph/{ceph.conf,ceph.client.admin.keyring}  node03.ceph.com:/etc/ceph/
```

iscsi-gw 节点配置iscsi-gateway.cf

```shell
vim /etc/ceph/iscsi-gateway.cfg 


# This is seed configuration used by the ceph_iscsi_config modules

# when handling configuration tasks for iscsi gateway(s)

# Please do not change this file directly since it is managed by Ansible and will be overwritten

[config]
cluster_name = ceph
gateway_keyring = ceph.client.admin.keyring

pool = rbd
cluster_client_name = client.admin
minimum_gateways = 1
fqdn_enabled = true
# API settings.

# The API supports a number of options that allow you to tailor it to your

# local environment. If you want to run the API under https, you will need to

# create cert/key files that are compatible for each iSCSI gateway node, that is

# not locked to a specific node. SSL cert and key files *must* be called

# 'iscsi-gateway.crt' and 'iscsi-gateway.key' and placed in the '/etc/ceph/' directory

# on *each* gateway node. With the SSL files in place, you can use 'api_secure = true'

# to switch to https mode.

# To support the API, the bear minimum settings are:

api_secure = False

# Optional settings related to the CLI/API service

api_user = admin
api_password = admin
api_port = 5000
loop_delay = 1

# 每个iscsi网关上的ip地址列表，用于管理操作，如目标创建，LUN导出等；trusted_ip_list可与用于iSCSI数据的ip相同，但条件允许时推荐使用分离的IP

trusted_ip_list = 172.16.120.158,172.16.120.159,172.16.120.160
```

#### 启动iscsi-gateway api服务

```shell
# 注意提前创建”rbd” pool
systemctl daemon-reload
systemctl enable rbd-target-api
systemctl start rbd-target-api
systemctl status rbd-target-api ; systemctl status rbd-target-gw

```



#### 创建iscsi-target与rbd image

Target-api: 实际上flask框架上做的 reset api ;为二次开发提供了全面的接口。

``curl  --user admin:admin -X PUT http://127.0.0.1:5000/api/target/iqn.2003-01.com.redhat.iscsi-gw``

Target-gw:  是Target-api 网关，提供给 cli 对 api 的控制

iscsi-gateway命令行工具gwcli用于创建/配置iscsi-target与rbd image；其余较低级别命令行工具，如targetcli或rbd等，可用于查询配置，但不能用于修改gwcli所做的配置。

- gwcli

```shell
o- / [root@R02-P01-BGW-002 ~]# gwcli 
/> ls
..

```

- 创建target

```shell
# 在iscsi-target目录下创建iscsi-target；
# iscsi-target命名规则：iqn.yyyy-mm.<reversed domain name>:identifier，即iqn.年-月.反转域名:target-name，这里没有域名，采用ip地址替代；
# 在新创建的iscsi-target下，同步生成gateway，host-groups，hosts目录
/> cd /iscsi-target 
/iscsi-target> create target_iqn=iqn.2020-04.com.redhat.iscsi-gw:ceph-igw

```

- 创建iscsi-gateway

```shell
/iscsi-target> cd iqn.2020-04.com.redhat.iscsi-gw:ceph-igw/gateways
/iscsi-target...i-gw/gateways> create gateway_name=R02-P01-BGW-002 ip_address=172.16.1.62 skipchecks=true
/iscsi-target...i-gw/gateways> create gateway_name=R02-P01-BGW-001 ip_address=172.16.1.61 skipchecks=true


```

- 创建rbd image

```shell
/iscsi-target...-igw/gateways> cd /disks/
/disks> create pool=rbd image=disk03 size=1G
ok
/disks> ls
```

- 设置initiator

```shell
/disks> cd /iscsi-target/iqn.2020-04.com.redhat.iscsi-gw:ceph-igw/hosts
/iscsi-target...scsi-gw/hosts> create client_iqn=iqn.2020-04.com.redhat.iscsi-client:iscsi-initiator
ok
# 设置CHAP认证（必须），否则iscsi-target会拒绝initiator的登陆请求；
# 在新建的initiator-name目录下设置认证
/iscsi-target...csi-initiator> auth chap=iscsiname/iscsipassword
ok
```

- 添加image 到initiator

```shell
/iscsi-target...csi-initiator> disk add rbd.disk01
ok
/iscsi-target...csi-initiator> ls

# 在新建的initiator-name目录下向initiator添加image；
# 添加成功后，对应initiator下有可被挂载的lun设备；
# 此时多台iscsi-gateway主机iscsi-gateway ip的tcp 3260端口被监听

[root@R02-P01-BGW-001 ~]# ss -anlt|grep ':3260 '
LISTEN     0      256    172.16.1.61:3260                     *:*                  
LISTEN     0      256    172.16.1.62:3260                     *:*                  

```

修改和删除target服务

```shell
[root@R02-P01-BGW-001 ~]#  rados ls -p rbd
rbd_header.764576b8b4567
gateway.conf  # iscsigw 配置文件
rbd_directory
rbd_id.disk03
rbd_header.8a0d12a3c18af
rbd_info
rbd_object_map.764576b8b4567
rbd_id.disk01
rbd_header.89fb06b8b4567
rbd_object_map.89fb06b8b4567
rbd_object_map.8a0d12a3c18af
rbd_id.disktemp

# 导出配置文件
[root@R02-P01-BGW-001 ~]# rados -p rbd  get gateway.conf  iscsigw.json 

{
    "clients": {},
    "created": "2020/04/15 05:29:22",
    "disks": {
    },
    "epoch": 12,
    "gateways": {
    },
    "groups": {},
    "updated": "2020/04/20 07:58:38",
    "version": 3
}
# 修改完后导入集群
[root@R02-P01-BGW-001 ~]# rados -p rbd  put gateway.conf  iscsigw.json 
[root@R02-P01-BGW-001 ~]# systemctl restart rbd-target-api.service 
```



```
cat `find /sys/ -name alua_access_type` 
for i in `find /sys/ -name alua_access_type`    ;do echo 1 > $i;done

```



### Iscsi-initiator客户端

- 安装initiator与multipath工具

```shell
# iscsi-initiator-utils是通用initiator套件；
# device-mapper-multipath是多路径工具
yum install iscsi-initiator-utils device-mapper-multipath -y 

```

- 配置multipath服务

```shell
# 启用multipath服务，生成”/etc/multipath.conf”文件
mpathconf --enable --with_multipathd y

# 在”/etc/multipath.conf”文件新增配置，针对LIO后端存储设置多路径ha
[root@node03 ~]# cat /etc/multipath.conf

defaults {
        user_friendly_names yes
        path_grouping_policy multibus
        failback immediate
        no_path_retry fail
}
devices {
        device {
                vendor                 "LIO-ORG"
                hardware_handler       "1 alua"
                path_grouping_policy   "failover"
                path_selector          "queue-length 0"
                failback               60
                path_checker           tur
                prio                   alua
                prio_args              exclusive_pref_bit
                fast_io_fail_tmo       25
                no_path_retry          queue
        }
}

```

- 设置chap认证

```shell

# 开启initiator的chap认证，并设置username/password，与iscsi-target设置保持一致；
# CHAP Settings部分， 
[root@ceph-client ~]# vim /etc/iscsi/iscsid.conf
node.session.auth.authmethod = CHAP
node.session.auth.username = iscsiname
node.session.auth.password = iscsipassword 

```

- 设置initiatoe-name

```shell
# 设置initiator-name，保持与iscsi-target设置的initiator-name一致

[root@localhost ~]# vim /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2020-04.com.redhat.iscsi-client:iscsi-initiator 

# 重启一下
[root@localhost ~]# systemctl restart   iscsid  
```

- 发现目标，登入挂在

```shell
[root@localhost ~]# iscsiadm --mode discovery --type st --portal 172.16.1.61
172.16.1.62:3260,1 iqn.2003-01.com.redhat.iscsi-gw
172.16.1.61:3260,2 iqn.2003-01.com.redhat.iscsi-gw
[root@localhost ~]# iscsiadm --mode node
172.16.1.62:3260,1 iqn.2003-01.com.redhat.iscsi-gw
172.16.1.61:3260,2 iqn.2003-01.com.redhat.iscsi-gw
[root@localhost ~]# iscsiadm --mode session 
iscsiadm: No active sessions.

[root@localhost ~]# iscsiadm --mode node --login -T iqn.2003-01.com.redhat.iscsi-gw
Logging in to [iface: default, target: iqn.2003-01.com.redhat.iscsi-gw, portal: 172.16.1.62,3260] (multiple)
Logging in to [iface: default, target: iqn.2003-01.com.redhat.iscsi-gw, portal: 172.16.1.61,3260] (multiple)
Login to [iface: default, target: iqn.2003-01.com.redhat.iscsi-gw, portal: 172.16.1.62,3260] successful.
Login to [iface: default, target: iqn.2003-01.com.redhat.iscsi-gw, portal: 172.16.1.61,3260] successful.

[root@localhost ~]# iscsiadm --mode session 
tcp: [1] 172.16.1.62:3260,1 iqn.2003-01.com.redhat.iscsi-gw (non-flash)
tcp: [2] 172.16.1.61:3260,2 iqn.2003-01.com.redhat.iscsi-gw (non-flash)


[root@localhost ~]# multipath -ll
mpathd (36001405bfc863062f1e437b98feb8076) dm-4 LIO-ORG ,TCMU device     
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
`-+- policy='queue-length 0' prio=50 status=active
  `- 10:0:0:1 sdf 8:80 active ready running
mpathc (36001405007f3b65aa144cd5a483e0cb7) dm-3 LIO-ORG ,TCMU device     
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
`-+- policy='queue-length 0' prio=50 status=active
  `- 10:0:0:2 sde 8:64 active ready running
mpathb (36001405ec9e5640783645708d5c6c409) dm-2 LIO-ORG ,TCMU device     
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
`-+- policy='queue-length 0' prio=50 status=active
  `- 10:0:0:0 sdd 8:48 active ready running
```

- 推出iscsi挂在

```shell
[root@localhost ~]# iscsiadm --mode node --logout
[root@localhost ~]# iscsiadm --mode node --op delete -T iqn.2003-01.com.redhat.iscsi-gw:ceph-gw
[root@localhost ~]# iscsiadm --mode node 
iscsiadm: No records found
```

- 格式化网络磁盘挂在本地

```shell
vgcreate vg0 /dev/mapper/mpathi 
lvcreate -l 100%FREE -n lv0 vg0
mkfs.xfs /dev/mapper/vg0-lv0 
mkdir /data/app/target -p
mount /dev/mapper/vg0-lv0  /data/app/target/
```

#### NFS 共享服务端

```
[root@localhost ~]# cat /etc/exports
/data/app/target 172.16.120.0/24(rw,insecure,sync,no_root_squash,fsid=123)

[root@localhost ~]# exportfs -rv
```

#### NFS 客户端挂载

```
[root@localhost ~]#  showmount -e 172.16.120.178
Export list for 172.16.120.178:
/data/app/target 172.16.120.0/24

[root@localhost ~]#  mount -t nfs4 172.16.120.176:/data/app/target/  /mnt/cache-nfs
```

开机自动挂载

```
[root@localhost ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Wed Feb 13 10:12:22 2019
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=1a4432c3-31a2-4d04-af01-5eb9e6ed4ad6 /boot                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
# 开机挂载
/dev/mapper/vg0-lv0   /data/app/target          xfs     defaults,_netdev 0 0
```



### Vmware 虚拟机导入KVM

```shell
root@pve01:/tmp/jj-vm# ls
cacheVMware-disk1.vmdk  cacheVMware.mf  cacheVMware.ovf

root@pve01:/tmp/jj-vm# qm importovf 144 cacheVMware.ovf vms --format 
qcow2  raw    vmdk   
root@pve01:/tmp/jj-vm# qm importovf 144 cacheVMware.ovf vms --format raw
```



### 开启监控mgr dashboard

[参考](https://www.cnblogs.com/passzhang/p/12179816.html)

开启dashboard 功能

```shell
ceph mgr module enable dashboard
# 关闭监控 ceph mgr module disable dashboard
```

创建证书

```shell
ceph dashboard create-self-signed-cert
```

创建 web 登录用户密码

```shell
ceph dashboard set-login-credentials user-name password
```

查看服务访问方式

```shell
ceph mgr services
```

在/etc/ceph/ceph.conf中添加

```shell
[mgr]
mgr_modules = dashboard
```


设置dashboard的ip和端口[可选]

```shell
ceph config-key put mgr/dashboard/server_addr 172.16.1.21
ceph config-key put mgr/dashboard/server_port 7000
```

### cephfs文件系统

```shell
root@pve02:~# pveceph mds create --hotstandby 1 --name pve02
creating MDS directory '/var/lib/ceph/mds/ceph-pve02'
creating keys for 'mds.pve02'
setting ceph as owner for service directory
enabling service 'ceph-mds@pve02.service'
Created symlink /etc/systemd/system/ceph-mds.target.wants/ceph-mds@pve02.service -> /lib/systemd/system/ceph-mds@.service.
starting service 'ceph-mds@pve02.service'
```

部署mds节点

```
 yum install ceph-mds-13.2.8
 sudo -u ceph  mkdir -p /var/lib/ceph/mds/ceph-iscsi01  #ceph-${cephfs_id}
 sudo ceph auth get-or-create mds.iscsi01 mon 'profile mds' mgr 'profile mds' mds 'allow *' osd 'allow *' > /var/lib/ceph/mds/ceph-iscsi01/keyring
 
 sudo systemctl start ceph-mds@iscsi01
 sudo systemctl enable ceph-mds@iscsi01
 ceph osd pool create cephfs_metadata 128 128
 ceph osd pool create cephfs_data  128 128
 ceph fs new cephfs cephfs_metadata  cephfs_data
 
  ceph fs status
  
```

fuse 挂载

```shell
yum install ceph-fuse-13.2.8
ceph auth get-or-create client.cephfs mon 'allow r' mds 'allow' osd 'allow rw pool=cephfs_metadata, allow rw pool=cephfs_data' -o /etc/ceph/ceph.client.cephfs.keyring

ceph auth get client.cephfs

ceph-fuse  -k /etc/ceph/ceph.client.cephfs.keyring  -n client.cephfs /mnt/mycephfs/
# 开机挂载
echo "id=cephfs,keyring=/etc/ceph/ceph.client.cephfs.keyring,ceph.conf=/etc/ceph/ceph.conf /mnt/mycephfs fuse.ceph defaults,_netdev 0 0 " >> /etc/fstab
# system 服务方式挂载
systemctl start ceph-fuse@-mnt-mycephfs.service
```

内核挂载

```shell
mount -t ceph 172.16.120.158:6789:/  /mnt/mycephfs -o name=admin,secret=AQChi5BemR29DxAA1pcVNBrsVb1rfRDDCGX8Sg==

#或者用 secretfile=/etc/ceph/ceph.client.cephfs.keyring

# 添加到/etc/fstab 
# {ipaddress}:{port}:/ {mount}/{mountpoint} {filesystem-name}     [name=username,secret=secretkey|secretfile=/path/to/secretfile],[{mount.options}]
172.16.120.158:6789:/     /mnt/ceph    ceph   name=cephfs,secretfile=/etc/ceph/secret.key,noatime,_netdev    0       0

```



### 找不到相应的pgid

[官方解决方案](https://docs.ceph.com/docs/mimic/rados/troubleshooting/troubleshooting-pg/)

```shell
# 告警现象
[store@R02-P02-MN-001.nm.cn ~]$ sudo ceph health detail
 HEALTH_WARN Reduced data availability: 1 pg stale; Degraded data redundancy: 1 pg undersized
 PG_AVAILABILITY Reduced data availability: 1 pg stale
     pg 4.2 is stuck stale for 67636.244121, current state stale+active+undersized, last acting [1591,1110]
 PG_DEGRADED Degraded data redundancy: 1 pg undersized
     pg 4.2 is stuck undersized for 67984.718711, current state stale+active+undersized, last acting [1591,1110]
     
# 解决办法   
[store@R02-P02-MN-001.nm.cn ~]$ sudoceph pg 4.2 query
Error ENOENT: i don't have pgid 4.2
# 由于没有找到这个pid 可能是集群重启导致的丢失。
[store@R02-P02-MN-001.nm.cn ~]$ sudo ceph pg dump_stuck unclean
ok
PG_STAT STATE                   UP          UP_PRIMARY ACTING      ACTING_PRIMARY 
4.2     stale+active+undersized [1591,1110]       1591 [1591,1110]           1591

[store@R02-P02-MN-001.nm.cn ~]$ sudo ceph osd force-create-pg 4.2 --yes-i-really-mean-it
pg 4.2 now creating, ok
```



### 安全移除故障osd

#### 1.查看集群健康状态

```shell
ceph -s
ceph -w # 动态
ceph osd tree #osd树
```

#### 2.分步移除

将osd从crush中删除，并删除对应的auth,host

##### 2.1把OSD剔除集群

**回到mon**

标记osd 权重为0

```shell
[root@R02-P01-MN-001 ~]# ceph osd out osd.9
osd.9 is already out.
 9   hdd  7.29999         osd.9              up        0 1.00000 #权重为0 
[root@R02-P01-MN-001 ~]# ceph -w
....
  io:
    recovery: 7.8 MiB/s, 2 objects/s   #看到在迁移数据
```

##### 2.2停服务

**到dn**

```shell
[root@R02-P01-DN-003 ~]# systemctl stop ceph-osd@9
```

##### 

##### 2.3删除crush图对应的osd条目

它就不再接收数据了

```
[root@R02-P01-MN-001 ~]# ceph osd crush rm osd.9 
removed item id 9 name 'osd.9' from crush map
[root@R02-P01-MN-001 ~]# ceph osd tree
ID CLASS WEIGHT   TYPE NAME               STATUS REWEIGHT PRI-AFF 
-1       51.09991 root default                                    
-2       21.89996     host R02-P01-DN-001                         
 0   hdd  7.29999         osd.0               up  1.00000 1.00000 
 1   hdd  7.29999         osd.1               up  1.00000 1.00000 
 2   hdd  7.29999         osd.2               up  1.00000 1.00000 
-4       21.89996     host R02-P01-DN-002                         
 3   hdd  7.29999         osd.3               up  1.00000 1.00000 
 4   hdd  7.29999         osd.4               up  1.00000 1.00000 
 5   hdd  7.29999         osd.5               up  1.00000 1.00000 
-3        7.29999     host R02-P01-DN-003                         
 8   hdd  7.29999         osd.8               up  1.00000 1.00000 
 9              0 osd.9                     down        0 1.00000 #已经移除对应的host
```

##### 2.4 删除osd认证密钥,删除osd.4

```
[root@R02-P01-MN-001 ~]# ceph auth rm osd.9
updated

[root@R02-P01-MN-001 ~]#  ceph osd rm osd.9
removed osd.9
[root@R02-P01-MN-001 ~]# ceph osd tree     
ID CLASS WEIGHT   TYPE NAME               STATUS REWEIGHT PRI-AFF 
-1       51.09991 root default                                    
...
-3        7.29999     host R02-P01-DN-003                         
 8   hdd  7.29999         osd.8               up  1.00000 1.00000 
```

#### 3.一步移除

##### 3.1 停服务

到dn节点

```
[root@R02-P01-DN-003 ~]#  systemctl stop ceph-osd@8
```

到mon节点

##### 3.2 移除osd

```shell
[root@R02-P01-MN-001 ~]#  ceph osd purge osd.8 --yes-i-really-mean-it
 ceph osd treepurged osd.8
[root@R02-P01-MN-001 ~]#  ceph osd tree
ID CLASS WEIGHT   TYPE NAME               STATUS REWEIGHT PRI-AFF 
-1       43.79993 root default                                    
...
-4       21.89996     host R02-P01-DN-002                         
 3   hdd  7.29999         osd.3               up  1.00000 1.00000 
 4   hdd  7.29999         osd.4               up  1.00000 1.00000 
 5   hdd  7.29999         osd.5               up  1.00000 1.00000 
-3              0     host R02-P01-DN-003          # 可以将主机从crush 实图删除 

[root@R02-P01-MN-001 ~]# ceph osd crush rm  R02-P01-DN-003
removed item id -3 name 'R02-P01-DN-003' from crush map

[root@R02-P01-MN-001 ~]#  ceph osd tree                   
ID CLASS WEIGHT   TYPE NAME               STATUS REWEIGHT PRI-AFF 
-1       43.79993 root default                                    
-2       21.89996     host R02-P01-DN-001                         
 0   hdd  7.29999         osd.0               up  1.00000 1.00000 
 1   hdd  7.29999         osd.1               up  1.00000 1.00000 
 2   hdd  7.29999         osd.2               up  1.00000 1.00000 
-4       21.89996     host R02-P01-DN-002                         
 3   hdd  7.29999         osd.3               up  1.00000 1.00000 
 4   hdd  7.29999         osd.4               up  1.00000 1.00000 
 5   hdd  7.29999         osd.5               up  1.00000 1.00000
```

#### 4. 清理故障osd 占用的磁盘

```shell
[root@R02-P01-DN-003 ~]# wipefs -a /dev/sde
wipefs: error: /dev/sde: probing initialization failed: Device or resource busy
[root@R02-P01-DN-003 ~]# dmsetup status
ceph--aeaeba1f--5281--4908--a52a--334350492163-osd--block--ae3e5e55--8240--438b--83a3--e1df0ea7dc95: 0 41934848 linear 
ceph--17e06781--df68--4720--a99e--c09f6832b48d-osd--block--d5ab223b--9c9a--44c9--a189--932e31e80933: 0 41934848 linear 
# 找到vg 串号对应的 osd data 盘
# ls -l /var/lib/ceph/osd/ceph-*
[root@R02-P01-DN-003 ~]# dmsetup remove ceph--aeaeba1f--5281--4908--a52a--334350492163-osd--block--ae3e5e55--8240--438b--83a3--e1df0ea7dc95

# 擦除磁盘
[root@R02-P01-DN-003 ~]# wipefs -a /dev/sde

```

#### 5.手动添加osd

 (prepare-->activate == create)

```shell
ceph-volume --cluster ceph lvm prepare --bluestore --data /dev/sdd --osd-id 6 --osd-fsid `uuidgen` --block.db /dev/sdg1 --block.wal /dev/sdg2
ceph-volume lvm activate --all #或者指定osdid fsid

# create 命令包含prepare和activate两个命令，执行完成后会立即让osd进入集群，分别使用前两个命令可以避免数据立即均衡
ceph-volume --cluster ceph lvm create --bluestore --data /dev/sdb --osd-id 7 --osd-fsid `uuidgen` --block.db /dev/sde1 --block.wal /dev/sde2 	
#将osd 加入主机视图
# ceph osd crush add `osd.id` `weihgt` host=`hostname -s` 
ceph osd crush add osd.7  7.3  host=`hostname -s` 
```



### 新增DN节点

#### 1.初始化 DN 节点

##### 1.1编辑资产文件 

- xstore-ansible-v2/etc/hosts
- /etc/hosts 解析
- 分发解析

```
[root@R02-P01-TBJ-001 xstore-ansible-v2]# cat hosts_add 
[center]
R02-P01-TBJ-001.zj.cn
[mons]
R02-P01-MN-00[1:3].zj.cn
[proxies]
R02-P01-PXY-00[1:3].zj.cn

[osds]
R02-P01-DN-00[1:3].zj.cn
# 新增dn节点
R02-P01-DN-004.zj.cn 

[others]
R02-P01-FGW-00[1:3].zj.cn

[all:vars]
ansible_ssh_user=root
ansible_ssh_pass=fufu

```

```
ansible -i hosts_add 'all:!center' -m copy -a 'src=/etc/hosts dest=/etc/hosts'
```

##### 1.2执行初始化剧本

指定新增主机

```
[root@R02-P01-TBJ-001 xstore-ansible-v2]# ansible-playbook  -i hosts_add  site.yml  --limit R02-P01-DN-004.zj.cn --list-host

playbook: site.yml

  play #1 (all): all    TAGS: []
    pattern: [u'all']
    hosts (1):
      R02-P01-DN-004.zj.cn
      
[root@R02-P01-TBJ-001 xstore-ansible-v2]# ansible-playbook  -i hosts_add  site.yml  --limit R02-P01-DN-004.zj.cn 
```

Play 执行的内容

```
TASK [copy /etc/hosts to others] 
TASK [ceph-init : get center host'ip] 
TASK [ceph-init : disable iptables] 
TASK [ceph-init : create backup directory] 
TASK [ceph-init : backup repo file] 
TASK [ceph-init : copy internal.repo to each host] 
TASK [ceph-init : rename hosts] 
TASK [ceph-init : check groups] 
TASK [ceph-init : create users] 
TASK [ceph-init : free sudo] 
TASK [ceph-init : copy ssh key] 
TASK [ceph-init : sync time] 
TASK [ceph-init : set timezone] 
TASK [ceph-init : server sync time] 
TASK [ceph-init : sync ntpd client time] 
TASK [ceph-init : sync ntpd client time in crontab] 
TASK [ceph-init : copy ssh_config.sh to each host] 
TASK [ceph-init : execute ssh_config.sh] 
TASK [execute init_system.sh to each host] 
```

#### 2. 部署DN节点

初始化结束后就可以把DN 节点加入集群

##### 2.1 编辑资产文件

```
[store@R02-P01-TBJ-001 ceph-cluster-deploy]$ cat hosts_osd
[osds]
R02-P01-DN-00[1:3].zj.cn
R02-P01-DN-004.zj.cn

[mons]
R02-P01-MN-001.zj.cn
R02-P01-MN-002.zj.cn
R02-P01-MN-003.zj.cn

```

##### 2.2 环境变量/格式化磁盘

执行两个脚本(可以加到剧本里)

```shell
 ansible -i hosts_osd 'R02-P01-DN-004.zj.cn'  -m script -a "echo_bashrc.sh" 
 ansible -i hosts_osd 'R02-P01-DN-004.zj.cn' -m  script -a "dd_disk.sh"
 ansible -i hosts_osd 'R02-P01-DN-004.zj.cn' -m script -a "parted.sh" -b
```

##### 2.3 执行剧本

```shell
ansible-playbook -i hosts_osd site_osds.yml --limit R02-P01-DN-004.zj.cn --list-host
ansible-playbook -i hosts_osd site_osds.yml --limit R02-P01-DN-004.zj.cn

sudo ceph osd tree 

ID  CLASS WEIGHT   TYPE NAME               STATUS REWEIGHT PRI-AFF 
 -9              0 host R02-P01-MN-001                             
 -1       87.59985 root default                                    
 -2       21.89996     host R02-P01-DN-001                         
  0   hdd  7.29999         osd.0               up  1.00000 1.00000 
  1   hdd  7.29999         osd.1               up  1.00000 1.00000 
  2   hdd  7.29999         osd.2               up  1.00000 1.00000 
 -4       21.89996     host R02-P01-DN-002                         
  3   hdd  7.29999         osd.3               up  1.00000 1.00000 
  4   hdd  7.29999         osd.4               up  1.00000 1.00000 
  5   hdd  7.29999         osd.5               up  1.00000 1.00000 
 -3       21.89996     host R02-P01-DN-003                         
  6   hdd  7.29999         osd.6               up  1.00000 1.00000 
  7   hdd  7.29999         osd.7               up  1.00000 1.00000 
  8   hdd  7.29999         osd.8               up  1.00000 1.00000 
-11       21.89996     host R02-P01-DN-004                         
  9   hdd  7.29999         osd.9               up  1.00000 1.00000 
 10   hdd  7.29999         osd.10              up  1.00000 1.00000 
 11   hdd  7.29999         osd.11              up  1.00000 1.00000 
```



### 新增MON节点

#### 1. 初始化MON节点

```shell
# 操作同初始化 DN 节点
```

#### 2. 部署MON 节点

##### 2.1编辑资产文件

```shell
[store@R02-P01-TBJ-001 ceph-cluster-deploy]$ cat hosts
[mons]
R02-P01-MN-001.zj.cn
R02-P01-MN-002.zj.cn
R02-P01-MN-003.zj.cn
# 新增MON
R02-P01-MN-004.zj.cn
  
[mgrs]
R02-P01-MN-001.zj.cn
R02-P01-MN-002.zj.cn
R02-P01-MN-003.zj.cn
# 新增MON
R02-P01-MN-004.zj.cn
  
[rgws]
R02-P01-FGW-00[1:3].zj.cn

```

##### 2.2 执行剧本

```shell
ansible-playbook -i hosts site.yml  --limit mons --list-host
ansible-playbook -i hosts site.yml  --limit mons 
```

**检查集群健康**

```shell
[store@R02-P01-MN-004 ~]$ sudo ceph -s 
  cluster:
    id:     2e2bbdf3-adad-488d-88c6-02ac38f2d439
    health: HEALTH_OK
 
  services:
    mon: 4 daemons, quorum R02-P01-MN-001.zj.cn,R02-P01-MN-002.zj.cn,R02-P01-MN-003.zj.cn,R02-P01-MN-004.zj.cn
    mgr: R02-P01-MN-001.zj.cn(active), standbys: R02-P01-MN-003.zj.cn, R02-P01-MN-002.zj.cn, R02-P01-MN-004.zj.cn
    osd: 12 osds: 12 up, 12 in
    rgw: 3 daemons active
 
  data:
    pools:   6 pools, 1280 pgs
    objects: 142  objects, 326 MiB
    usage:   97 GiB used, 227 GiB / 324 GiB avail
    pgs:     1280 active+clean
```

### 新增FGW 节点

#### 1. 初始化FGW节点

##### 1.1 编辑资产文件

```shell
# 操作同初始化 DN 节点
# 编辑/etc/hosts ，xstore-ansible-v2/hosts 新增 R02-P01-FGW-004.zj.cn
[root@R02-P01-TBJ-001 xstore-ansible-v2]# cat hosts_add 
[center]
R02-P01-TBJ-001.zj.cn
[mons]
R02-P01-MN-00[1:3].zj.cn
R02-P01-MN-004.zj.cn
[proxies]
R02-P01-PXY-00[1:3].zj.cn

[osds]
R02-P01-DN-00[1:3].zj.cn
R02-P01-DN-004.zj.cn

[others]
R02-P01-FGW-00[1:3].zj.cn
#新增节点
R02-P01-FGW-004.zj.cn

[center:vars]
ansible_ssh_user=root
ansible_ssh_pass=fufu
ansible_ssh_port=22

[all:vars]
ansible_ssh_user=root
ansible_ssh_pass=fufu
```

##### 1.2 拷贝解析

```shell
ansible -i hosts_add 'all:!center' -m copy -a 'src=/etc/hosts dest=/etc/hosts'
```

##### 1.3 执行剧本

**指定新增主机**

```
ansible-playbook  -i hosts_add  site.yml  --limit R02-P01-FGW-004.zj.cn --list-host  
ansible-playbook  -i hosts_add  site.yml  --limit R02-P01-FGW-004.zj.cn 

```

#### 2. 部署FGW节点

##### 2.1 修改资产文件

```shell
[store@R02-P01-TBJ-001 ceph-cluster-deploy]$ cat hosts
[mons]
R02-P01-MN-001.zj.cn
R02-P01-MN-002.zj.cn
R02-P01-MN-003.zj.cn
R02-P01-MN-004.zj.cn
  
[mgrs]
R02-P01-MN-001.zj.cn
R02-P01-MN-002.zj.cn
R02-P01-MN-003.zj.cn
R02-P01-MN-004.zj.cn
  
[rgws]
R02-P01-FGW-00[1:3].zj.cn
#新增节点
R02-P01-FGW-004.zj.cn
```

##### 2.2 执行剧本

```shell
# 可以指定主机 --limit R02-P01-FGW-004.zj.cn
ansible-playbook -i hosts site.yml  --limit rgws --list-host
ansible-playbook -i hosts site.yml  --limit rgws 
```



## rancher

### 安装daocker

```shell
curl https://releases.rancher.com/install-docker/19.03.sh | sh
```



### 国内镜像源配置

```shell
cat >/etc/docker/daemon.json <<eof
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://docker.mirrors.ustc.edu.cn",
    "https://reg-mirror.qiniu.com",
    "https://hub-mirror.c.163.com/"
  ]
}
eof
# 配置http_proxy【可选】

  mkdir -p /etc/systemd/system/docker.service.d
cat >/etc/systemd/system/docker.service.d/http-proxy.conf<<eof
[Service]
Environment="HTTPS_PROXY=http://192.168.56.153:808/" "HTTP_PROXY=http://192.168.56.153:808/" "NO_PROXY=localhost,127.0.0.0/8,172.17.0.0/8"
eof

sudo systemctl daemon-reload
sudo systemctl restart docker


{
  "registry-mirrors": [
  "https://reg-mirror.qiniu.com",
  "https://mirror.ccs.tencentyun.com",
  "https://dockerhub.azk8s.cn",
  "https://docker.mirrors.ustc.edu.cn/"
  ]
}
```



## Ansible

```shell
ansible -i hosts all -m lineinfile -a  "dest=/etc/sudoers state=present regexp='^store ALL=' line='store ALL=(ALL) NOPASSWD: ALL' " 
```

