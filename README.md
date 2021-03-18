

<h1 align = "center">Bind-DLZ + Flask  + Mysql  DNS管理平台 </h1>

系统环境:CentOS 7.2 X64

软件版本: 

      bind-9.9.5.tar.gz  
      MariaDB-devel-10.1.48-1.el7.centos.x86_64
      MariaDB-common-10.1.48-1.el7.centos.x86_64
      MariaDB-server-10.1.48-1.el7.centos.x86_64
      MariaDB-shared-10.1.48-1.el7.centos.x86_64
      MariaDB-client-10.1.48-1.el7.centos.x86_6
描述： 
```
MariaDB数据库安装配置yum 源
[root@docker-host01 local]# cat /etc/yum.repos.d/mariadb.repo
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

#安装MariaDB
[root@docker-host01 local]#yum install -y MariaDB-server MariaDB-client MariaDB-devel mysql-libs
```

<h2 align = "center">一．源码安装配置Bind: </h2>

1.源码编译安装

	 tar -zxvf  bind-9.9.5.tar.gz           #解压压缩包
	 cd bind-9.9.5
	 ./configure --prefix=/usr/local/bind/  \
	 --enable-threads=no \
	 --enable-newstats   \
	 --with-dlz-mysql    \
	 --disable-openssl-version-check
	 
     #官网说明强调编译关闭多线程，即--enable-threads=no
	 
     make
	 make install           #源码编译安装完成

 
2.环境变量配置

	cat >>  /etc/profile  <<EOF 
	PATH=$PATH:/usr/local/bind/bin:/usr/local/bind/sbin
	export  PATH
	EOF

	 source  /etc/profile  #重新加载一下环境变量
	 named -v           #如下图，说明环境变量正常


	 
![](https://github.com/1032231418/doc/blob/master/images/1.png?raw=true)


3.用户添加授权目录

	 useradd  -s  /sbin/nologin  named
	 chown  -R named:named /usr/local/bind/





4.配置Bind
```
 vi /usr/local/bind/etc/named.conf
# Start of rndc.conf
key "rndc-key" {
	algorithm hmac-md5;
	secret "SG8Xqevr9JI8F7UVqddZlQ==";
};
controls {
        inet 127.0.0.1 port 953
        allow { 127.0.0.1; } keys { "rndc-key"; };
};


options {
        tcp-clients 50000;
        directory "/usr/local/bind/var";
        pid-file "/usr/local/bind/var/bind.pid";
        dump-file "/usr/local/bind/var/bind_dump.db";
        statistics-file "/usr/local/bind/var/bind.stats";
        notify yes;
        recursion yes;
        version "ooxx-bind:1.0.24";
        allow-notify       { none; };
        allow-recursion    { any; };
        allow-transfer     { none; };
        allow-query        { any; };
};

logging {
        channel bind_log {
                file "/usr/local/bind/log/bind.log" versions 3 size 20m;
                severity info;
                print-time yes;
                print-severity yes;
                print-category yes;
        };
        category default {
                bind_log;
        };
};

view "breaklinux" {
      match-clients           {any; };
      allow-query-cache           {any; };
      allow-recursion          {any; };
      allow-transfer          {any; };
      dlz "mysql-dlz" {
              database "mysql
              {host=127.0.0.1 dbname=bind ssl=false port=3306 user=bind pass=123456}
              {select zone from dns_records where zone='$zone$'}
             {select ttl, type, mx_priority, case when lower(type)='txt' then concat('\"', data, '\"') when lower(type) = 'soa' then concat_ws(' ', data, resp_person, serial, refresh, retry, expire, minimum) else data end from dns_records where zone = '$zone$' and host = '$record$'}";
      };
};

保存退出
```
生成 name.ca文件

	(demo) -bash-4.1# cd /usr/local/bind/etc/
	(demo) -bash-4.1# dig -t NS .  >named.ca

5.配置数据库，导入sql 文件

	 mysql -p   #登录数据库
	mysql> CREATE DATABASE  named   CHARACTER SET utf8 COLLATE utf8_general_ci; 
	mysql> source named.sql;             #注意路径，这里我放在当前目录
	就两张表，一个dns用到的表，一个用户管理表

![](https://github.com/1032231418/doc/blob/master/images/2.png?raw=true)


6.启动  Bind 服务并设置开机启动脚本
```
root@docker-host01 local]#cat /usr/lib/systemd/system/named.service
[Unit]
Description=Internet domain name server
After=network.target

[Service]
ExecStart=/usr/local/bind/sbin/named -f -u named -4
ExecReload=/usr/local/bind/sbin/rndc reload
ExecStop=/usr/local/bind/sbin/rndc stop

[Install]
WantedBy=multi-user.target
Alias=bind.service

#设置开机启动
systemctl enable named.service

#启动DNS服务
systemctl start named.service
```
监控系统日志：

	 tail -f /var/log/messages
	 
如下，说明服务启动正常

![](https://github.com/1032231418/doc/blob/master/images/3.png?raw=true)

	测试bind连接数据库是否正常:

![](https://github.com/1032231418/doc/blob/master/images/4.png?raw=true)


 tail -f /var/log/messages

![](https://github.com/1032231418/doc/blob/master/images/5.png?raw=true)

<h2 align = "center">二．配置Bind-Web 管理平台 </h2>

上传 Bind-web-1.0.tar.gz 管理平台

	(demo) -bash-4.1# git  clone  https://github.com/breaklinux/Bind-Web.git  #git  克隆下来
	(demo) -bash-4.1# cd Bind-Web
	(demo) -bash-4.1# python  run.py     

运行软件程序使用flask框架写的，要用pip安装该框架

http://ip/5000   访问WEB 界面 登录账户 eagle 密码 123456

功能有，用户管理，域名管理

![](https://github.com/1032231418/doc/blob/master/images/6.png?raw=true)


![](https://github.com/1032231418/doc/blob/master/images/7.png?raw=true)

![](https://github.com/1032231418/doc/blob/master/images/8.png?raw=true)				

![](https://github.com/1032231418/doc/blob/master/images/jiexi.png?raw=true)

