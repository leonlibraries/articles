---
title: CDH安装运维小记
date: 2017-04-13 10:27:15
tags: [大数据,CDH,Hadoop,HBase]
categories: 大数据
---

### 理想中的部署蓝图
![Deploy](deploy.jpeg)

### Hosts 文件

这里主要记录我在安装过程中遇到的问题以及解决的办法，跟着文档一步一步走的东西不多赘述。

我这边部署六台服务器(系统版本 CentOS 6.5)，如下所示
```
192.168.240.100 CDHt-240-100
192.168.240.101 CDHt-240-101
192.168.240.102 CDHt-240-102
192.168.240.103 CDHt-240-103
192.168.240.104 CDHt-240-104
192.168.240.105 CDHt-240-105
```
也需要将上面的代码段植入到**每台服务器**的/etc/hosts文件中

### Build Local Yum Repository

如果网络不好的，建议在本地配置好 yum 源，参考文档 https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_yumrepo_local_create.html
里边包含了 Hadoop 相关的所有包，然而后续我打算用 Parcels 安装，因此这里的包并非全部用得上。但是我们可以丰富这里的包，因为在安装过程中有些包的安装耗时是非常大的。这里把我知道的包列举出来，尽量不在网络传输的过程中浪费太多时间。

我把 Yum 源配置在``CDHt-240-102``这台机子。接着 cloudera-manager-server 也要装在这台机子上。之后要用到 Parcels 安装，这种安装方式最大的好处就是一次下载多地安装，我们可以先这么干。
```sh
mkdir -p /opt/cloudera/parcel-repo
cd /opt/cloudera/parcel-repo
wget https://archive.cloudera.com/cdh5/parcels/5.10/manifest.json
wget https://archive.cloudera.com/cdh5/parcels/5.10/CDH-5.10.1-1.cdh5.10.1.p0.10-el6.parcel.sha1
wget https://archive.cloudera.com/cdh5/parcels/5.10/CDH-5.10.1-1.cdh5.10.1.p0.10-el7.parcel
mv CDH-5.10.1-1.cdh5.10.1.p0.10-el6.parcel.sha1 CDH-5.10.1-1.cdh5.10.1.p0.10-el6.parcel.sha
```
另外还有一些JDK安装包以及 cloudera-manager 安装组件，也可以先下载下来
```sh
# 这个地址是 lighttpd 的目录，上边有给出相关文档
cd /var/www/lighttpd/cdh/5/RPMS/x86_64
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/cloudera-manager-agent-5.10.1-1.cm5101.p0.6.el6.x86_64.rpm
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/cloudera-manager-daemons-5.10.1-1.cm5101.p0.6.el6.x86_64.rpm
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/cloudera-manager-server-5.10.1-1.cm5101.p0.6.el6.x86_64.rpm
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/cloudera-manager-server-db-2-5.10.1-1.cm5101.p0.6.el6.x86_64.rpm
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/enterprise-debuginfo-5.10.1-1.cm5101.p0.6.el6.x86_64.rpm
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/jdk-6u31-linux-amd64.rpm
wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5/RPMS/x86_64/oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm
```
还有 MySQL 驱动包
```sh
wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.41.tar.gz
tar zxvf mysql-connector-java-5.1.41.tar.gz

sudo cp mysql-connector-java-5.1.41/mysql-connector-java-5.1.41-bin.jar /usr/share/java/mysql-connector-java.jar

# 这个地址是 lighttpd 对外界开通的网络目录，其他机器也是需要这个驱动包的
sudo cp mysql-connector-java-5.1.41/mysql-connector-java-5.1.41-bin.jar /var/www/lighttpd/mysql/mysql-connector-java.jar

```
光是等待下载这些包就要花不少时间。
随后就可以将集群中所有机器的 yum 源配置一下，在``/etc/yum.repos.d``目录下创建文件``cloudera-cdh5-local.repo``，内容为
```txt
[cloudera-cdh5-local]
# Packages for Cloudera's Distribution for Hadoop, Version 5, on RedHat or CentOS 6 x86_64
name=Cloudera's Distribution for Hadoop, Version 5
baseurl=http://192.168.240.102/cdh/5/
gpgkey =https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/RPM-GPG-KEY-cloudera
gpgcheck = 1
```

Well done.
### Install MySQL 5.7
MySQL 我也部署在了``CDHt-240-102``上，具体部署过程参考
https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/

```sh
wget https://repo.mysql.com//mysql57-community-release-el6-10.noarch.rpm
sudo rpm -Uvh mysql57-community-release-el6-10.noarch.rpm

yum repolist enabled | grep mysql
sudo yum install mysql-community-server

sudo service mysqld start
```
5.7版本 MySQL 加强了密码安全机制，root 初始化密码很复杂，通常在 mysql 的启动日志里有，如果日志被清理了那就得参考这个文档了
https://dev.mysql.com/doc/refman/5.7/en/resetting-permissions.html

安装完，确保 root 用户可用后，需要根据文档预先创建一些库，且配置好 MySQL 相关配置项及驱动，参考如下
https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_installing_configuring_dbs.html#cmig_topic_5
https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_mysql.html#id_ijy_cwt_g5

至此，准备工作才算基本做完。

### Cloudera Manager Server & Cloudera Manager Agent
真正的安装步骤现在才开始，请参考这篇官方说明，一步一步走
https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_b.html#cmig_topic_6_6_3
会发现有了上面的准备过程后，接下来的安装都非常顺利和迅速。

Cloudera Manager Server 安装完后就可以进入管理界面，默认端口7180

这个时候可以让所有机器执行如下脚本：
```
#!/bin/sh

useradd cloudera-scm

# usermod -a -G sudo cloudera-scm

sudo mkdir -p /var/lib/hadoop-httpfs
sudo mkdir -p /var/lib/oozie
sudo mkdir -p /var/lib/sqoop2
sudo mkdir -p /var/lib/solr
sudo mkdir -p /var/data
sudo mkdir -p /cm
sudo mkdir -p /cm/var/log/zookeeper/
sudo mkdir -p /cm/var/lib/zookeeper/version-2
sudo mkdir -p /cm/var/log/hadoop-hdfs/

sudo chown cloudera-scm:cloudera-scm /var/lib/hadoop-httpfs
sudo chown cloudera-scm:cloudera-scm /var/lib/oozie
sudo chown cloudera-scm:cloudera-scm /var/lib/sqoop2
sudo chown cloudera-scm:cloudera-scm /var/lib/solr
sudo chown cloudera-scm:cloudera-scm /var/data
sudo chown cloudera-scm:cloudera-scm /cm
sudo chown cloudera-scm:cloudera-scm /cm/var/log/zookeeper/
sudo chown cloudera-scm:cloudera-scm /cm/var/lib/zookeeper/version-2/
sudo chown cloudera-scm:cloudera-scm /cm/var/log/hadoop-hdfs/

sudo chmod 755 /var/lib/hadoop-httpfs
sudo chmod 755 /var/lib/oozie
sudo chmod 755 /var/lib/sqoop2
sudo chmod 755 /var/lib/solr
sudo chmod 755 /var/data
sudo chmod 755 /cm
sudo chmod 755 /cm/var/log/zookeeper/
sudo chmod 755 /cm/var/lib/zookeeper/version-2/
sudo chmod 755 /cm/var/log/hadoop-hdfs/

# touch /cm/var/lib/zookeeper/myid

sed -i '$a\session		required	pam_limits.so' /etc/pam.d/su
sed -i '$a\%cloudera-scm ALL=(ALL) NOPASSWD: ALL' /etc/sudoers
sed -i '$a\vm.swappiness=10' /etc/sysctl.conf

# echo never > /sys/kernel/mm/transparent_hugepage/defrag 以禁用此设置，然后将同一命令添加到 /etc/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/defrag
sed -i '$a\echo never > /sys/kernel/mm/transparent_hugepage/defrag' /etc/rc.local

# 我在内网配置了 mysql 驱动包的 HTTP 访问
wget 'http://192.168.240.102/mysql/mysql-connector-java.jar'

cp mysql-connector-java.jar /usr/share/java/
cp mysql-connector-java.jar /var/lib/oozie/

rm -f mysql-connector-java.jar

reboot
```
### 单用户模式
执行完脚本自动重启，包括 MySQL 和 Cloudera Manager Server 也会自动重启回来，接来下就可以利用 Parcels 进行集群安装和部署了，当然这个过程是可视化的。
这里我使用的是单用户模式做集群部署的，也可以参考如下文档https://www.cloudera.com/documentation/enterprise/latest/topics/install_singleuser_reqts.html#concept_o5q_stg_2v
，文档所涉及的上边的脚本有覆盖，因此不需要做相关动作了。
![CDH](CDH.jpeg)


### 安装过程遇到的问题

##### 单用户模式启动遇到的问题

单用户安装过程在最后一步启动 zookeeper 的时候，会报出一个错误```cloudera KeyError: 'getpwnam(): name not found: zookeeper'```以至于后续组件无法初始化和启动。然而实际上在仪表盘上启动 zk 集群是能成功的，这就很纠结了。
这个报错的意思是 zookeeper 的初始化和安装过程需要一个叫「zookeeper」的用户去做，然而系统里找不到这个用户，然而我用的是单用户模式，理应这个用户只有可能是「cloudera-scm」，因此我暂时将其归结为一个 Bug。
我怎么去避开这个 Bug 呢？
其实很简单，先安装 zookeeper 集群，手动在仪表盘里启动集群，启动成功后，一步一步添加其他组件（HDFS、Hive、Spark、HBase、Yarn 等）集群搭建完成。就这么简单。


##### agent 无法启动且没有任何报错日志
这种情况特别诡异，直到我查到了这么一个issue
https://github.com/mattshma/bigdata/issues/65
其实 agent 启动不了大多数和 hostname 有关，我之前的 hostname 命名包含了下划线，实际上用起来没问题，但是不符合规则的。

>The Internet standards (Requests for Comments) for protocols mandate that component hostname labels may contain only the ASCII letters 'a' through 'z' (in a case-insensitive manner), the digits '0' through '9', and the hyphen ('-'). The original specification of hostnames in RFC 952, mandated that labels could not start with a digit or with a hyphen, and must not end with a hyphen. However, a subsequent specification (RFC 1123) permitted hostname labels to start with digits. No other symbols, punctuation characters, or white space are permitted.
While a hostname may not contain other characters, such as the underscore character (\_), other DNS names may contain the underscore.[4] Systems such as DomainKeys and service records use the underscore as a means to assure that their special character is not confused with hostnames. For example, \_http.\_sctp.www.example.com specifies a service pointer for an SCTP capable webserver host (www) in the domain example.com. Note that some applications (e.g. Microsoft Internet Explorer) won't work correctly if any part of the hostname contains an underscore character.[5]

正确的 hostname 命名由 ``a-zA-z``,``0-9``和``-``组成。
这也是个坑，找不到任何报错日志，就是启动不了 agent。如果不 google 一下恐怕难以解决问题。

##### sudo: sorry, you must have a tty to run sudo
出现这个问题是跟你的 cloudera-scm 账户权限有关，解决办法是修改 ``/etc/sudoers``文件

将 ``Defaults requiretty`` 修改为 ``Defaults !requiretty`` 即可！
### 通过 Parcels 升级组件
![upgrade1](upgrade1.jpeg)
![upgrade2](upgrade2.jpeg)
当然，这个 parcel 后缀的包也是可以用``wget``命令预先下载到``/opt/cloudera/parcel-repo``这个目录下的。

done.
