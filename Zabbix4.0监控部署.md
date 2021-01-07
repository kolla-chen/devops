## 1.安装MySQL

部署文档：https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/

```shell
~]# yum -y install yum-utils 
~]# rpm -ivh https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
~]# yum-config-manager --disable mysql80-community
~]# yum-config-manager --enable mysql57-community
~]# yum install mysql-community-server
~]# systemctl start mysqld
~]# systemctl status mysqld
~]# grep 'temporary password' /var/log/mysqld.log
~]# mysql -uroot -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Zabbix2020!';
```

优化mysql配置

```shell
~]# cat > /etc/my.cnf <<eof
[mysql]
socket = /var/lib/mysql/mysql.sock
[mysqld]
user = mysql
port = 3306
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
bind-address = 0.0.0.0
pid-file = /var/run/mysqld/mysqld.pid
character-set-server = utf8
collation-server = utf8_general_ci
log-error = /var/log/mysqld.log

max_connections = 10240
open_files_limit = 65535
innodb_buffer_pool_size = 1G
innodb_flush_log_at_trx_commit = 2
innodb_log_file_size = 256M
eof
```

## 2.安装php

```shell
安装依赖包:
~]# yum install -y gcc gcc-c++ make gd-devel libxml2-devel \
libcurl-devel libjpeg-devel libpng-devel openssl-devel \
libxslt-devel

安装PHP:
~]# wget http://mirrors.sohu.com/php/php-5.6.36.tar.gz
~]# tar zxf php-5.6.36.tar.gz
~]# cd php-5.6.36
~]# ./configure --prefix=/usr/local/php \
--with-config-file-path=/usr/local/php/etc \
--enable-fpm --enable-opcache \
--with-mysql --with-mysqli  \
--enable-session --with-zlib --with-curl --with-gd \
--with-jpeg-dir --with-png-dir --with-freetype-dir \
--enable-mbstring --enable-xmlwriter --enable-xmlreader \
--enable-xml --enable-sockets --enable-bcmath --with-gettext
~]# make -j 8 && make install
~]# cp php.ini-production /usr/local/php/etc/php.ini
~]# cp sapi/fpm/php-fpm.conf /usr/local/php/etc/php-fpm.conf
~]# cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
~]# cp sapi/fpm/php-fpm.service /usr/lib/systemd/system/
~]# cat > /usr/lib/systemd/system/php-fpm.service <<eof
[Unit]
Description=The PHP FastCGI Process Manager
After=syslog.target network.target

[Service]
Type=simple
PIDFile=/usr/local/php/var/run/php-fpm.pid
ExecStart=/usr/local/php/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
eof

~]# systemctl daemon-reload
~]# systemctl start php-fpm
~]# systemctl enable php-fpm

```

## 3.安装nginx

```shell
~]# wget http://nginx.org/download/nginx-1.9.9.tar.gz
~]# yum install gcc pcre-devel openssl-devel –y
~]# useradd -M -s /sbin/nologin nginx
~]# tar -xf nginx-1.9.9.tar.gz 
~]# cd nginx-1.9.9
~]# ./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module
~]# make && make install
~]# chown nobody -R /usr/local/nginx/
~]# vi /usr/local/nginx/conf/nginx.conf
~]#id        /var/run/nginx.pid;

~]# cat > /usr/lib/systemd/system/nginx.service<<eof
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
eof

~]# systemctl daemon-reload

```

## 4.部署Zabbix Server

```shell
https://www.zabbix.com/download_sources

导入表结构：
~]# cd /opt/src/zabbix-4.0.0/database/mysql/
~]# mysql -uroot -p
mysql> use zabbix
mysql> source schema.sql;
mysql> source images.sql;
mysql> source data.sql;

```

```shell
~]# yum install libxml2-devel libcurl-devel libevent-devel net-snmp-devel mysql-community-devel java-1.8.0-openjdk java-1.8.0-openjdk-devel -y
~]# useradd  -M zabbix -s /sbin/nologin
~]# cd zabbix-4.0.0
~]# ./configure --prefix=/usr/local/zabbix --enable-server --enable-agent --enable-java --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2
~]# make install

# vi /usr/local/zabbix/etc/zabbix_server.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=Zabbix2018!

# cat > /usr/lib/systemd/system/zabbix_server.service<<eof
[Unit]
Description=Zabbix Server
After=syslog.target
After=network.target

[Service]
Environment="CONFFILE=/usr/local/zabbix/etc/zabbix_server.conf"
EnvironmentFile=-/etc/sysconfig/zabbix-server
Type=forking
Restart=on-failure
PIDFile=/tmp/zabbix_server.pid
KillMode=control-group
ExecStart=/usr/local/zabbix/sbin/zabbix_server -c $CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
TimeoutSec=0

[Install]
WantedBy=multi-user.target
eof


启动Agent：
# /usr/local/zabbix/sbin/zabbix_agentd
启动server
# systemctl daemon-reload
# systemctl restart zabbix_server
```

## 部署Zabbix Web界面

```shell
Zabbix前端使用PHP写的，所以必须运行在PHP支持的Web服务器上。
# cp zabbix-4.0.0/frontends/php/* /usr/local/nginx/html/ -rf
```

