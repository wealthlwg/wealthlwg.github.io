#           CDH5.16.1集群部署（基于联通沃云）  
**本部署参考以下几篇文章进行部署：**  
[Deploying CDH 5 on a Cluster](https://docs.cloudera.com/documentation/enterprise/5-6-x/topics/cdh_ig_cdh5_cluster_deploy.html)  
[CDH安装前置准备 ](https://mp.weixin.qq.com/s?__biz=MzI4OTY3MTUyNg==&mid=2247485512&idx=1&sn=9e953a7eb8b3b2a64a011550ab7da184&chksm=ec2ad841db5d51573f5913d14c33135180bca023de1c349fc431f561c055d1d085527107b66e&scene=21#wechat_redirect)  
[如何在Redhat7.4安装CDH5.16.1 ](https://cloud.tencent.com/developer/article/1377153)  
[Cloudera Manager 离线安装](https://javinjunfeng.top/technicalstack/cm/385)  

## 1. 基本介绍
### 1.1 主机信息
**主机配置信息：**  
* 主机数量：4台（1台master + 3台worker)
* 资源配置：16Core CPU + 64G Memory + 500G HDD
* 操作系统：CentOS 7.6

**各主机部署设计：**

|序号|原主机名|新主机名|IP|部署内容|
|:--:|:--:|:--:|:--:|:--:|
|1|hadoop-82|master-01|10.15.8.82|HTTPD、MariaDB、CM、CM Server、Coudera Manager、NameNode Manager、Zookeeper、Kudu master|
|1|hadoop-82|master-01|10.15.8.82|DataNode、CM、NameNode Manager、Zookeeper、Kudu master|
|3|hadoop-84|worker-02|10.15.8.84|DataNode、CM、NodeManager|
|4|hadoop-85|worker-03|10.15.8.85|DataNode、CM、NodeManager|  

\*全程以root权限进行部署

## 2. 部署之前准备
### 2.1 升级系统包
* 为保持系统最新包，需要更新系统包
```C
[root@hadoop-82 ~]# yum -y update
已加载插件：fastestmirror, langpacks Loading mirror speeds from cached hostfile
file:///media/repodata/repomd.xml: [Errno 14] curl#37 - "Couldn't open file /media/repodata/repomd.xml"
正在尝试其它镜像。
No packages marked for update
```
* 根据报错信息，打开·/etc/yum.repos.d/iso.repo·    
```shell
[root@hadoop-83 downloads]# cat /etc/yum.repos.d/iso.repo
[base]
name=CentOS-$releasever - Base
baseurl=file:///media/
gpgcheck=0
enabled=1
```
* 根据`baseurl=file:///media/`查找`/media/`目录，发现目录下为空，估计运维漏掉。  
没yum源文件就得填一个，国内163的yum源用得比较多：  
```shell
[root@hadoop-82 ~]# wget -O base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo  
[root@hadoop-82 ~]# mv /etc/yum.repo.d/iso.repo /etc/yum.repo.d/iso.repo.bak  
[root@hadoop-82 ~]# mv base.repo /etc/yum.repo.d/  
[root@hadoop-82 ~]# yum clean all  
[root@hadoop-82 ~]# yum makecache  
[root@hadoop-82 ~]# yum -y update  
```
一共更新1245个包，其他三台主机(10.15.8.83, 10.15.8.84, 10.15.8.85)同样操作之。  

### 2.2 修改主机名
* 修改本机主机名  
```shell
[root@hadoop-82 data]# vim /etc/hostname
[root@hadoop-82 data]# cat /etc/hostname
master-01
```
* 修改`/etc/hosts`，添加所有主机信息进去  
```shell
[root@hadoop-82 data]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.15.8.82 master-01
10.15.8.83 worker-01
10.15.8.84 worker-02
10.15.8.85 worker-03
```
* 重启  
```shell
[root@hadoop-82 ~]# reboot
```
\*其他三台同样操作，重启后四台主机名已改变。

### 2.3 集群主机间免密登录  
* 产生RSA密钥对  
```shell
[root@master-01 ~]# ssh-keygen  
```

* 公钥复制到其所有四台主机（没看错，包含自己）
```shell
[root@master-01 ~]# ssh-copy-id worker-01 -p 22088
```
* 验证登录到worker-01主机  
```shell
[root@master-01 ~]# ssh -p 22088 worker-01
Last login: Mon Mar  2 12:16:07 2020 from 10.40.206.195
```
可以看到已经OK，同样将publickey复制到master-01(本机)、worker-02、worker-03主机，并验证之。 
 
**\*注意：每一台机器都要产生密钥对，并将产生的公钥复制到所有主机。**

### 2.4 创建批处理脚本  
集群主机有许多共性操作，比如上面的更新源操作，关闭防火墙，禁用SELinux等等，每台主机都单独配置太麻烦，得用上批处理脚本。  
* 批量处理脚本：ssh_do_all.sh  
```shell
#! /bin/bash
for n in `cat $1`
do
ssh -p 22088 -t $n $2
done
```
* 批量传送文件脚本：bk_cp.sh  
```shell
#!/bin/bash

if [ $# -lt 3 ];
then
  echo "Usage: $0 <node-list-file> <source-file> <target-file>"
  exit
fi

serverList=$1
file=$2
target=$3

for server in $(cat $serverList)
do
  scp -P 22088 -r $file $server:$target
done
```

* 所有主机列表：node.list
```shell
master-01 # 主节点
worker-01 # 工作节点1
worker-02 # 工作节点2
worker-03 # 工作节点3
```

* 所有工作节点：node_other.list
```shell
worker-01 # 工作节点1
worker-02 # 工作节点2
worker-03 # 工作节点3
```

### 2.5 磁盘检测  
* 检测磁盘无坏区  
```shell
[root@master-01 ~]# badblocks -v /dev/sdb
正在检查从 0 到 524287999的块
Checking for bad blocks (read-only test):
done
Pass completed, 0 bad blocks found. (0/0/0 errors)
```
* DataNode数据盘确保没有配置分区卷LogicalVolume Manager(LVM)  
```shell
[root@master-01 ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0   50G  0 disk
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   49G  0 part
  ├─centos-root 253:0    0 45.1G  0 lvm  /
  └─centos-swap 253:1    0  3.9G  0 lvm  [SWAP]
sdb               8:16   0  500G  0 disk
└─vgdata-lvdata 253:2    0  500G  0 lvm  /data
sr0              11:0    1 1024M  0 rom
```
以上模型显示sdb为单一分区。

### 2.6 网络配置
* 禁止IPV6，因为当前Hadoop还不支持  
将以下内容补充到`/etc/sysctl.conf` 
```shell
net.ipv6.conf.all.disable_ipv6= 1
net.ipv6.conf.default.disable_ipv6= 1
net.ipv6.conf.lo.disable_ipv6= 1
```
将以下内容补充到`/etc/sysconfig/network`  
```shell
vim /etc/sysconfig/network
NETWORKING_IPV6=no
IPV6INIT=no
```

* 设置静态IP  
查看当前ip信息，看是否有` BOOTPROTO=static`语句，如没有则加进去  
```shell
[root@master-01 ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens192
TYPE=Ethernet
BOOTPROTO=static
NAME=ens192
DEVICE=ens192
ONBOOT=yes
IPADDR=10.15.8.82
GATEWAY=10.15.8.94
NETMASK=255.255.255.224
DNS1=116.116.116.116
```
这是申请的云主机服务器，肯定有static IP,所以其实可以直接Pass这一步。  

## 3. 开始部署

### 3.1 关闭防火墙  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh  node.list "systemctl stop firewalld"
```
* 开机不启动防火墙
```shell
[root@master-01 scripts]# sh ssh_do_all.sh  node.list "systemctl disable firewalld"
```
查看防火墙状态  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh  node.list "systemctl status firewalld"
```
### 3.2 禁用SELinux  
* 批处理执行`setenforce 0`命令  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "setenforce 0"
```
* 集群所有节点修改`/etc/selinux/config`文件如下：  
```shell
SELINUX=disabled
SELINUXTYPE=targeted
```
* 检查所有机器是否修改成功  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "cat /etc/selinux/config | grep SELINUX"
```
### 3.3 集群时钟同步  
chrony为Centos7默认安装程序，是NTP(Network Time Protocal)的一个通用版本，其作用是通过与NTP服务器通讯实现时间同步，
在这里先卸载chrony，再安装NTP。使用NTP配置各主机的时间同步，将master主机(10.15.8.82)作为NTP服务器，同步时间到其他三台主机。  
* 所有机器卸载chrony  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "yum -y remove chrony"
```
以上只列一台主机信息。

* 安装NTP  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "yum -y install ntp"
```
以上为一台主机信息。

* master机器配置时钟与自己同步  
主机的`/etc/ntp.conf`19行左右作以下修改  
```shell
server  127.127.1.0     # local clock
fudge   127.127.1.0 stratum 10
```
* 其他集群主机时间配置与同步
其它节点配置`/etc/ntp.conf`19行左右    
```shell
server  10.15.8.82     # cm clock
```
* 重启所有机器的ntp服务  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "systemctl restart ntpd"
```
* 查看各主机状态
以下为master的ntp状态
```shell
[root@master-01 scripts]#  sh ssh_do_all.sh node.list "systemctl status ntpd"
```
* 激活ntp服务
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "systemctl enable ntpd"
```
* 验证同步结果  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "ntpq -p"
```
左边出现*号表示同步成功。


### 3.4 设置swap  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "echo vm.swappiness = 10 >> /etc/sysctl.conf"
[root@master-01 scripts]# sh ssh_do_all.sh node.list "sysctl vm.swappiness=10"
```

### 3.5 禁用透明大页面(Centos7默认开启）
* 将开启命令写进配置文件  
```shell
[root@master-01 scripts]# sh ssh_do_all.sh node.list "echo never > /sys/kernel/mm/transparent_hugepage/defrag "
[root@master-01 scripts]# sh ssh_do_all.sh node.list  "echo never > /sys/kernel/mm/transparent_hugepage/enabled"
```
* 设置开机自动关闭  
将如下脚本添加到master下`/etc/rc.d/rc.local`文件中
```shell
[root@master-01 scripts]# vim /etc/rc.d/rc.local
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then 
echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi 
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then 
echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
```
文件同步到所有节点  
```shell
[root@master-01 scripts]# sh bk_cp.sh node.list /etc/rc.d/rc.local /etc/rc.d
```

### 3.6 安装http服务  
* 直接yum安装  
```shell
[root@master-01 ~]# yum -y install httpd
```
* 设置开机启动  
```shell
[root@master-01 download]# systemctl start httpd.service
[root@master-01 download]# systemctl enable httpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service
```
* 查看http服务状态
```shell
[root@master-01 ~]# systemctl status httpd.service
```

### 3.7 安装MariaDB  
* yum安装MariaDB
```shell
[root@master-01 ~]# yum -y install mariadb
[root@master-01 ~]# yum -y install mariadb-server
```
* 启动并配置MariaDB  
**设置root角色的密码为：MariaDB**
```shell
[root@master-01 ~]# systemctl start mariadb
[root@master-01 ~]# systemctl enable mariadb
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
[root@master-01 ~]# /usr/bin/mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password: MariaDB
Re-enter new password: MariaDB
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```  
### 3.8 建立CM，Hive等需要的表
* 登录数据库进行配置  
```shell
[root@master-01 ~]# mysql -u root -p
```
执行以下SQL语句以创建9个数据库:   
创建**hive**数据库   
> create database metastore default character set utf8;    
> CREATE USER 'hive'@'%' IDENTIFIED BY 'password';    
> GRANT ALL PRIVILEGES ON metastore. * TO 'hive'@'%';    
>  FLUSH PRIVILEGES;    

创建**cm**数据库  
> create database cm default character set utf8;    
> CREATE USER 'cm'@'%' IDENTIFIED BY 'password';     
> GRANT ALL PRIVILEGES ON cm. * TO 'cm'@'%';     
> FLUSH PRIVILEGES;    

创建**am**数据库  
> create database am default character set utf8;     
> CREATE USER 'am'@'%' IDENTIFIED BY 'password';      
> GRANT ALL PRIVILEGES ON am. * TO 'am'@'%';     
> FLUSH PRIVILEGES;    

创建**rm**数据库  
> create database rm default character set utf8;    
> CREATE USER 'rm'@'%' IDENTIFIED BY 'password';    
> GRANT ALL PRIVILEGES ON rm. * TO 'rm'@'%';     
> FLUSH PRIVILEGES;

创建**hue**数据库  
> create database hue default character set utf8;   
> CREATE USER 'hue'@'%' IDENTIFIED BY 'password';    
> GRANT ALL PRIVILEGES ON hue. * TO 'hue'@'%';   
> FLUSH PRIVILEGES;

创建**oozie**数据库  
> create database oozie default character set utf8;  
> CREATE USER 'oozie'@'%' IDENTIFIED BY 'password';    
> GRANT ALL PRIVILEGES ON oozie. * TO 'oozie'@'%';    
> FLUSH PRIVILEGES;

创建**sentry**数据库（暂时没用到）  
> create database sentry default character set utf8;    
> CREATE USER 'sentry'@'%' IDENTIFIED BY 'password';    
> GRANT ALL PRIVILEGES ON sentry. * TO 'sentry'@'%';    
> FLUSH PRIVILEGES;

创建**nav_ms**数据库  
> create database nav_ms default character set utf8;    
> CREATE USER 'nav_ms'@'%' IDENTIFIED BY 'password';   
> GRANT ALL PRIVILEGES ON nav_ms. * TO 'nav_ms'@'%';   
> FLUSH PRIVILEGES;

创建**nav_ns**数据库  
> create database nav_as default character set utf8;  
> CREATE USER 'nav_as'@'%' IDENTIFIED BY 'password';   
> GRANT ALL PRIVILEGES ON nav_as. * TO 'nav_as'@'%';  
> FLUSH PRIVILEGES;  

### 3.9 安装jdbc驱动  
* 下载jdbc并解压  
```shell
[root@master-01 download]# mkdir -p /usr/share/java/
[root@master-01 download]# wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.48.tar.gz
[root@master-01 download]# tar -zxvf mysql-connector-java-5.1.48.tar.gz
[root@master-01 download]# cd mysql-connector-java-5.1.48/
[root@master-01 mysql-connector-java-5.1.48]# mv mysql-connector-java-5.1.48.jar /usr/share/java/
[root@master-01 download]# cd /usr/share/java
[root@master-01 java]# ln -s /usr/share/java/mysql-connector-java-5.1.48.jar mysql-connector-java.jar
[root@master-01 java]# ll
. . .
lrwxrwxrwx  1 root root      47 3月   2 22:09 mysql-connector-java.jar -> /usr/share/java/mysql-connector-java-5.1.48.jar
. . .
```
* 同步到其他主机  
` [root@master-01 scripts]# sh bk_cp.sh node_other.list /usr/share/java /usr/share/`

## 4. Cloudera Manager安装
**下载各安装包之前，先在/data/repo/下面新建cm5.16.1和cdh5.16.1文件夹以便存放下载内容**  

### 4.1 安装CM/CDH以及配置源  
* 下载CM5.16.1的安装包（放于同一目录下）  
7个安装包地址为:   
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-agent-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm  
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-daemons-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm  
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-server-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm  
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-server-db-2-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm   
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/enterprise-debuginfo-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm   
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/jdk-6u31-linux-amd64.rpm   
> http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm   

先下载第一个  
```shell
[root@master-01 cm]# wget http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-agent-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
--2020-03-02 22:20:35--  http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-agent-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
正在解析主机 archive.cloudera.com (archive.cloudera.com)... 失败：未知的名称或服务。
```
DAMN IT! 解析域名失败...修改DNS为如下  
```shell
[root@master-01 cm]# vim /etc/resolv.conf
nameserver 8.8.8.8 #google域名服务器
nameserver 8.8.4.4 #google域名服务器
[root@master-01 cm]# wget -c http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-agent-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
```
可惜联通访问外网的速度差强人意，只能先下载到个人电脑，再上传到服务器!  
下载过程忽略...     
在个人电脑操作以下命令  
```shell
$ scp -P 22088 -r cm/ root@10.15.8.82:~/data/repo/cm5.16.1
```

* 下载CDH5.16.1的安装包  
三个包地址为：  
> http://archive.cloudera.com/cdh5/parcels/5.16.1/CDH-5.16.1-1.cdh5.16.1.p0.3-el7.parcel  
> http://archive.cloudera.com/cdh5/parcels/5.16.1/CDH-5.16.1-1.cdh5.16.1.p0.3-el7.parcel.sha1    
> http://archive.cloudera.com/cdh5/parcels/5.16.1/manifest.json    

与前面一样，先下载到个人电脑，再上传到服务器  
在个人电脑执行以下命令  
```shell
$ scp -P 22088 -r cdh/ root@10.15.8.82:~/data/repo
root@10.15.8.82's password:
CDH-5.16.1-1.cdh5.16.1.p0.3-el7.parcel        100% 2029MB   2.3MB/s   14:54
```

* 登录master01进入cm目录，生成rpm元数据  
```shell
[root@hadoop-82 ~]# cd /data/repo/cm5.16.1  
[root@master-01 cm5.16.1]# createrepo .
```
* 配置Web服务器
* 将上述cdh5.16.1/cm5.16.1目录移动到（由于数据盘位于/data下，故这里只创建软连接，而不是移动）/var/www/html目录下, 使得用户可以通过HTTP访问这些rpm包。  
```shell
[root@master-01 ~]# ln -s /data/repo/cm5.16.1 /var/www/html/cm5.16.1
[root@master-01 ~]# ln -s /data/repo/cdh5.16.1 /var/www/html/cdh5.16.1
```

* 验证Web服务是否正常：  

 ```shell
[root@master-01 download]# wget http://10.15.8.82/cm5.16.1
--2020-03-03 15:21:41--  http://10.15.8.82/cm5.16.1
正在连接 10.15.8.82:80... 已连接。
```

以上结果表明已能正常访问。  

* 制作Cloudera Manager的repo源  
```shell
[root@master-01 download]# cat /etc/yum.repos.d/cm.repo
[cmrepo]
name = cm_repo
baseurl = http://10.15.8.82/cm5.16.1
enable = true
gpgcheck = false
```

* 验证安装JDK  
```shell
[root@master-01 download]# yum -y install oracle-j2sdk1.7-1.7.0+update67-1
```

* 同步cm源到其他机器  
```shell
[root@master-01 scripts]# sh bk_cp.sh node_other.list /etc/yum.repos.d/cm.repo  /etc/yum.repos.d
```

### 4.2 安装Cloudera Manager Server

* 通过yum安装Cloudera Manager Server  
```shell
[root@master-01 download]# yum -y install cloudera-manager-server
```

* 初始化数据库  
```shell
[root@master-01 schema]# cd ~
[root@master-01 ~]# /usr/share/cmf/schema/scm_prepare_database.sh mysql cm cm password
JAVA_HOME=/usr/java/jdk1.7.0_67-cloudera
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
Executing:  /usr/java/jdk1.7.0_67-cloudera/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/usr/share/cmf/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
```

* 启动Cloudera Manager Server  
```bash
[root@master-01 ~]# systemctl start cloudera-scm-server
```

* 检测7180端口监听状况  
```shell
[root@master-01 ~]# netstat -lnpt | grep 7180
tcp        0      0 0.0.0.0:7180            0.0.0.0:*               LISTEN      71479/java
```

* 浏览器访问CM
http://10.15.8.82:7180/cmf/login
```shell
curl hadoop-82:7180/cmf/login
```

## 5. 安装CDH集群

### 5.1 CDH集群安装向导
* 1. admin/admin登录CM  
* 2. agree with license protocol, click continue  
* 3. click 60 days trial, clidk continue  
* 4. 搜索主机  
主机列表如下
|已扩展查询|主机名称 (FQDN)|IP 地址|当前受管|结果|
|:----------:|:----------:|:----------:|:----------------------------------------:|
|10.15.8.82|master-01|10.15.8.82|否| 主机准备就绪：0 毫秒响应时间。|
|10.15.8.83|worker-01|10.15.8.83|否| 主机准备就绪：1 毫秒响应时间。|
|10.15.8.84|worker-02|10.15.8.84|否| 主机准备就绪：1 毫秒响应时间。|
|10.15.8.85|worker-03|10.15.8.85|否| 主机准备就绪：0 毫秒响应时间。|  

* 5. 使用parcel选择，点击“更多选项”,点击“-”删除其它所有地址，输入http://10.15.8.82/cdh5.16.1，点击“保存更改”  
保存更改后，这时回到上个页面会看到我们之前准备好的http的CDH5.16.1的源，如果显示不出来，可能http源配置有问题，请参考前面步骤仔细进行检查。  

* 6.选择自定义存储库，输入cm的http地址  

* 7.  点击“继续”，进入下一步安装jdk  

* 8. 点击“继续”，进入下一步，默认多用户模式，不需要进行任何勾选  

* 9. 点击“继续”，进入下一步配置ssh账号密码
出错，只有master成功，三台worker都失败，原因如下：
```shell
. . .
正在检测 Cloudera Manager Server...
BEGIN host -t PTR 10.15.8.82
82.8.15.10.in-addr.arpa domain name pointer bogon.
END (0)
using bogon as scm server hostname
BEGIN which python
/usr/bin/python
END (0)
BEGIN python -c 'import socket; import sys; s = socket.socket(socket.AF_INET); s.settimeout(5.0); s.connect((sys.argv[1], int(sys.argv[2]))); s.close();' bogon 7182
Traceback (most recent call last):
File "<string>", line 1, in <module>
File "/usr/lib64/python2.7/socket.py", line 224, in meth
return getattr(self._sock,name)(*args)
socket.gaierror: [Errno -2] Name or service not known
END (1)
could not contact scm server at bogon:7182, giving up
waiting for rollback request
. . .
```
**原因分析1**
根据领导经验，考虑是否主机名的问题，立马将所有主机的/etc/hostname，以及/etc/hosts里面的名字全改一遍  
简单来说就是，将“-”改为“0”，新主机名字如下  
|旧主机名|新主机名|
|:---------:|:---------:|
|master-01|master001|
|worker-01|worker001|
|worker-02|worker002|
|worker-03|worker003|

修改完后所有主机重启。
再登录`http://10.15.8.82/cmf/login`试了一遍，结果一样  
再看回上面这段提示：  
```shell
could not contact scm server at bogon:7182, giving up
```
在这里被我们的master主机为啥会被解析为bogon......一脸懵逼，一顿搜索之后又是联通DNS的问题，联通会将所有保留的网络地址RFC1918( http://tools.ietf.org/html/rfc1918)都指向"bogon" 这个hostname，F**K!  
**解决方法**  
将所有主机DNS修改为8.8.8.8/8.8.4.4，重启scm服务，安装ok.  
* 接下来都是可视化操作，点击继续即可
* 集群设置选择【含Impala的内核】
* 选择自定义安装组件，选择以下组件：  
```shell
HDFS, Hive, Hue, Impala, Kudu, Oozie, Spark, YARN, ZooKeeper
```
这里没选择Kafka是因为要事先安装其他插件，后面单独安装。

### 5.2 安装其他组件  
* 首先将Java升级到8  
具体cdh5.16.1支持哪个版本的jdk要看
[官网](https://docs.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html#concept_ihg_vf4_j1b)  
官网推荐安装最新版本java为**1.8u181**，另外还有直接下载到服务器，解压两个包
```shell
[root@master001 java8]# ls
 jdk-8u181-linux-x64.tar.gz  jce_policy-8.zip
[root@master001 java8]# tar -zxf jdk-8u181-linux-x64.tar.gz
[root@master001 java8]# unzip jce_policy-8.zip
[root@master001 java8]# ls
 jdk-8u181-linux-x64.tar.gz  jdk1.8.0_181 jce_policy-8.zip  UnlimitedJCEPolicyJDK8
```
将`UnlimitedJCEPolicyJDK8`目录下所有文件复制到`jdk1.8.0_181/jre/lib/security/`  
```shell
[root@master001 java8]# cp UnlimitedJCEPolicyJDK8/* ./jdk1.8.0_181/jre/lib/security/
[root@master001 java8]# ls jdk1.8.0_181/jre/lib/security/ -l
总用量 196
-rw-r--r-- 1   10  143   4054 7月   7 2018 blacklist
-rw-r--r-- 1   10  143   1273 7月   7 2018 blacklisted.certs
-rw-r--r-- 1   10  143 114757 7月   7 2018 cacerts
-rw-r--r-- 1   10  143   2466 7月   7 2018 java.policy
-rw-r--r-- 1   10  143  41530 7月   7 2018 java.security
-rw-r--r-- 1   10  143     98 7月   7 2018 javaws.policy
-rw-r--r-- 1 root root   3035 3月   5 12:00 local_policy.jar
drwxr-xr-x 4   10  143   4096 7月   7 2018 policy
-rw-r--r-- 1 root root   7323 3月   5 12:00 README.txt
-rw-r--r-- 1   10  143      0 7月   7 2018 trusted.libraries
-rw-r--r-- 1 root root   3023 3月   5 12:00 US_export_policy.jar
```
将jdk1.8.0_131目录拷贝至/usr/java目录下  
```shell
[root@master001 java8]# cp -r jdk1.8.0_181/ /usr/java/
```
将`jdk1.8.0_181-cloudera`同步到集群所有主机  
```shell
[root@master001 scripts]# sh ssh_do_all.sh node.list "rm -r /usr/java/jdk1.7.0_67/"
[root@master001 scripts]# sh bk_cp.sh node_other.list /usr/java/jdk1.8.0_181-cloudera/ /usr/java/
[root@master001 scripts]# sh ssh_do_all.sh node.list "source /etc/profile"
```
* Cloudera Manager配置  
进入[cloudera manager主页](http://10.15.8.82:7180/cmf/index)  
> 选择【主机】下的【所有主机】子菜单
> > 【配置】
> > > 【筛选器】下的【高级】  
> > > 在右边的Java主目录填入<u> /usr/java/jdk1.8.0_181-cloudera/</u>(不要留有空格)  
回到CM主页根据页面提示重启相应服务   

* 安装StreamSets  
详细过程请看[【这里】](https://mp.weixin.qq.com/s/r7Z8RlAm7az4TMdxFzZs8w?)   

* 安装kafka3.0.0 以及 Spark2.2  
详细安装过程请看[【这里】](https://cloud.tencent.com/developer/article/1078391)   

* 升级到kafka4.0  
先下载以下三个文件放在`/data/repo/kafka4.0/`   

> http://archive.cloudera.com/kafka/parcels/4.0.0/KAFKA-4.0.0-1.4.0.0.p0.1-el7.parcel  
> http://archive.cloudera.com/kafka/parcels/4.0.0/KAFKA-4.0.0-1.4.0.0.p0.1-el7.parcel.sha1  
>  http://archive.cloudera.com/kafka/parcels/4.0.0/manifest.json  

将上述sha1重命名为sha   

下载以下文件放到`/opt/cloudera/csd/`  
> http://archive.cloudera.com/csds/kafka/KAFKA-1.2.0.jar   

编辑以下文件：  
> /opt/cloudera/parcels/KAFKA-4.0.0-1.4.0.0.p0.1/etc/kafka/conf.dist/server.properties   

修改内容如下：  

|hostname|broker.id|listeners|zookeeper.connection|  
|:--:|:--:|:--:|:--:|  
|worker001|1|worker001_broker://10.15.8.83:9092|10.15.8.83:2181,10.15.8.84:2181,10.15.8.85:2181|  
|worker002|1|worker002_broker://10.15.8.84:9092|10.15.8.83:2181,10.15.8.84:2181,10.15.8.85:2181|  
|worker003|1|worker003_broker://10.15.8.85:9092|10.15.8.83:2181,10.15.8.84:2181,10.15.8.85:2181|    

* 升级Spark到2.3  
同样地，从官网下载三个文件，并放在同一目录下：  

> http://archive.cloudera.com/spark2/parcels/2.3.0.cloudera2/SPARK2-2.3.0.cloudera2-1.cdh5.13.3.p0.316101-el7.parcel    
> http://archive.cloudera.com/spark2/parcels/2.3.0.cloudera2/SPARK2-2.3.0.cloudera2-1.cdh5.13.3.p0.316101-el7.parcel.sha1  
> http://archive.cloudera.com/spark2/parcels/2.3.0.cloudera2/manifest.json  

建立软连接以提供http访问   
> ln -s /data/repo/spark2.3/ /var/www/html/spark2.3  

browse访问http://master001/spark2.3，OK
另外下载对应csd文件  
> http://archive.cloudera.com/spark2/csd/SPARK2_ON_YARN-2.3.0.cloudera2.jar  

并将csd文件移到`/opt/cloudera/csd/`下  
在CM页面parcel管理，增加spark2.3源，并配置spark2.3地址为：`http://master001/spark2.3`   
安装完毕之后，在 master运行spark-shell命令会出现以下错误   
```shell
[root@master001 ~]# spark2-shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
20/03/08 11:35:07 ERROR spark.SparkContext: Error initializing SparkContext.
org.apache.hadoop.security.AccessControlException: Permission denied: user=root, access=WRITE, inode="/user":hdfs:supergroup:drwxr-xr-x
. . .
```

没有权限访问，我一个root角色居然没有权限！google大法一顿撸之后找到解决方法：  
对于/user，这个文件夹的拥有者不是所谓的“root”。实际上，这个文件夹为“hdfs”所有（755权限，这里将hdfs理解为一个属于supergroup的用户）。因此更改其权限为root。所以，你可以向这个文件夹随意的存、改文件了。  
> sudo -u hdfs hadoop fs -chown root /user   

验证修改结果:    
```shell
[root@master001 ~]# spark2-shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://master001:4040
Spark context available as 'sc' (master = yarn, app id = application_1583396341323_0012).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.0.cloudera2
      /_/

Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```
问题解决。 

来看看cm首页界面：  
![CM-HOME](https://master001-cm-home.oss-cn-beijing.aliyuncs.com/cm-home.png?Expires=1583719428&OSSAccessKeyId=TMP.hjjz4B3MajLMPq5up1Y7BAcQJHPNPaSv26eBebAc61aTEDgPZQmTmwnrKRE11TwrHBoChLKsfXNQDwuQEHsKuU42k2J6vx6fkeqymCJhEfTprBGxyt8q79hcpYnQ33.tmp&Signature=Uc4IkPj4p6HsfKBRVuAFikXQO0A%3D)
**至此，按要求部署CDH5.16.1完成，至于后面使用起来还有没坑，再补充（更新于2020/03/07）**  




