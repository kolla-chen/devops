## 批量升级Dell IDRAC固件

[**Dell CentOS 环境下安装远程管理命令racadm**](https://www.linuxidc.com/Linux/2019-08/159730.htm)

**[Dell racadm官方文档](https://www.dell.com/support/manuals/zh-cn/poweredge-c6420/idrac_4.00.00.00_racadm/fwupdate?guid=guid-d3e1b910-e30f-4fb0-a584-16fc7507fda1&lang=en-us)**



### 安装vsftpd

```shell
yum -y install vsftpd
systemctl start vsftpd
systemctl status vsftpd
# 加白带外网段
firewall-cmd  --add-source 172.32.25.0/24 --zone trusted 
```

### 下载dell IDRAC升级包

```shell
mv firmimg.d7 /var/ftp/pub/
chmod 755 /var/ftp/pub/*
```

### 编写idrac_getversion.sh

```shell
#!/bin/bash
ipmi_addr=172.32.25.
for ip in ${ipmi_addr}{1..20}
do
 echo "########$ip#####"
 sshpass -p "*****=" ssh root@${ip} -o StrictHostKeyChecking=no   racadm getversion
done

```

### 编写idrac_upgrade.sh

```shell
#!/bin/bash
ipmi_addr=172.32.25.
for ip in ${ipmi_addr}{1..20}
do
 echo "########$ip#####"
 sshpass -p "******=" ssh root@${ip} -o StrictHostKeyChecking=no racadm  fwupdate -f 172.32.25.202 anonymous 1  -d /pub/firmimg.d7
done
```

