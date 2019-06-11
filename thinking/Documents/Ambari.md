

# Ambari安装部署Hadoop集群

---
~~这几天一直弄Ambari部署，发现自己有很多不足的地方，特此记录。如有不妥之处，欢迎大家一起讨论研究。~~
 Ubuntu18按照官方安装Ambari2.7.3一直报各种问题，
 没办法只能试Ubuntu16.
 
 

---
***主服务机最好是60G哦！***

常用命令
```
    #正在进行的进程
   root@nm1-machine:~# ps -aef

    #端口使用情况
    netstat -ntlp
    
    #
```

```
#安装ambari-server成功后，每次启动系统，主从服务机都得开启服务
root@nm1-machine:/var/www/html# nohup python -m SimpleHTTPServer 1>out.log 2>&1 &
```


==下载链接==

[ambari-2.7.3.0 ](http://public-repo-1.hortonworks.com/ambari/ubuntu16/2.x/updates/2.7.3.0/ambari-2.7.3.0-ubuntu16.tar.gz) 

[HDP](http://public-repo-1.hortonworks.com/HDP/ubuntu16/3.x/updates/3.1.0.0/HDP-3.1.0.0-ubuntu16-deb.tar.gz)

[HDP-UTILS](http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/ubuntu16/HDP-UTILS-1.1.0.22-ubuntu16.tar.gz)

[HDP-GPL](http://public-repo-1.hortonworks.com/HDP-GPL/ubuntu16/3.x/updates/3.1.0.0/HDP-GPL-3.1.0.0-ubuntu16-gpl.tar.gz)


## 一、配置hosts、Selinux、THP、Databases

```
nm2@nm2-machine:~$ vi /etc/hosts #增加IP与域名对应
127.0.1.1       nm1-machine.com

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
192.168.159.245 nm2-machine.com
192.168.159.246 nm1-machine.com


nm2@nm2-machine:~$ vi /etc/hostname #修改机器名称
nm2@nm2-machine:~$ reboot #一定要重启系统

#查看 hostname 与hostname -A是否一致
nm2@nm2-machine:~$ hostname
nm2-machine.com
nm2@nm2-machine:~$ hostname -A
nm2.machine.com 


#使用sestatus -v 命令，查看Selinux状态。
root@nm1-machine:~# sestatus -v
root@nm1-machine:~# setenforce 0
root@nm1-machine:~# vi /etc/selinux/config 
SELINUX=disabled

root@nm1-machine:~# umask 0022
root@nm1-machine:~# umask
root@nm1-machine:~# echo umask 0022 >> /etc/profile


#在集群的每个节点，都要关闭。编辑/etc/grub.conf文件，在kernel 行后面加入transparent_hugepage=never，保存退出，重启机器。
transparent_hugepage=never


#为 Ranger配置数据库

root@nm1-machine:~# mysql -u root -p

mysql> CREATE USER 'rangerdba'@'localhost' IDENTIFIED BY 'rangerdba';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost';

mysql> CREATE USER 'rangerdba'@'%' IDENTIFIED BY 'rangerdba';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost' WITH GRANT OPTION;

mysql> GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%' WITH GRANT OPTION;

mysql> FLUSH PRIVILEGES;

mysql> exit



#在mysql中配置SAM和Schema 
root@nm1-machine:~# mysql -u root -p

mysql> create database registry;

mysql> create database streamline; 

mysql> CREATE USER 'registry'@'%' IDENTIFIED BY 'R12$%34qw';

mysql> CREATE USER 'streamline'@'%' IDENTIFIED BY 'R12$%34qw';

mysql> GRANT ALL PRIVILEGES ON registry.* TO 'registry'@'%' WITH GRANT OPTION ;

mysql> GRANT ALL PRIVILEGES ON streamline.* TO 'streamline'@'%' WITH GRANT OPTION ;

mysql> commit;



#在mysql中配置druid和supperset

mysql> CREATE DATABASE druid DEFAULT CHARACTER SET utf8;
mysql> CREATE DATABASE superset DEFAULT CHARACTER SET utf8;

mysql>  CREATE USER 'druid'@'%' IDENTIFIED BY 'druid';
mysql> CREATE USER 'superset'@'%' IDENTIFIED BY 'supperset';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'druid'@'%' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON *.* TO 'superset'@'%' WITH GRANT OPTION;

mysql> commit;



```

## 二、配置JDK

```
#我的openjdk-8不好使，只能手动安装jdk1.8
root@nm2-machine:/usr/local/software# unzip jdk-8u211-linux-x64.tar.gz.zip 

tar jdk-8u211-linux-x64.tar.gz 

#配置 环境变量 /etc/profile

# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).
JAVA_HOME=/usr/local/software/jdk1.8.0_211
CLASSPATH=$JAVA_HOME/lib/

PATH=$PATH:$JAVA_HOME/bin:
export PATH JAVA_HOME CLASSPATH 

#使/etc/profile生效
root@nm2-machine:/usr/local/software# source /etc/profile
#测试jdk是否安装成功
root@nm2-machine:/usr/local/software# javac -version
javac 1.8.0_211

```
## 三、SSH免密登录（主从服务机都弄，要不反复反分析能搞死你）

```
#先更新系统
root@nm1-machine:/home/nm1# sudo apt-get update

#安装*server的时候，请输入“Y”大写的
root@nm2-machine:/usr/local/software# apt-get install openssh-server
root@nm2-machine:/usr/local/software# apt-get install openssh-client

#默认是不让root登录的，我们需要修改ssh配置文件
root@nm2-machine:~# vi /etc/ssh/sshd_config 
#找到下面记录，改成下面这样。
# Authentication:
LoginGraceTime 120
**#PermitRootLogin prohibit-password**
*PermitRootLogin yes*
StrictModes yes

#重启ssh
root@nm2-machine:~# /etc/init.d/ssh restart

#验证是否成功
root@nm2-machine:~# ssh localhost

#在主机生成秘钥，一直默认Enter
root@nm1-machine:/home/nm1# ssh-keygen 

#将公钥追加到authorized_keys 文件中
root@nm1-machine:~# cat .ssh/id_rsa.pub >> .ssh/authorized_keys

#验证是否成功
ssh localhost

#在“从”服务机做以下操作(我的是nm2-machine)
scp root@192.168.159.246:/root/.ssh/id_rsa.pub /home/nm2

#把公钥追加到authorized_keys文件中，并删除本机生成的公钥
root@nm2-machine:~# cat /home/nm2/id_rsa.pub >> .ssh/authorized_keys
root@nm2-machine:~# rm /home/nm2/id_rsa.pub

#在“主（nm1-machine）”服务机测试是否可免密登录“从（nm2-machine）”服务机
root@nm1-machine:~/.ssh# ssh 192.168.159.245

```
## 四、配置环境

```
#创建目录
mkdir -p /var/www/html
ls /var/www/html
#把那四个包，放入当前目录下，并解压缩

#查看当前目录有以下四个文件夹
root@nm1-machine:/var/www/html# ls
ambari  HDP  HDP-3.1.0.0-ubuntu16-deb.tar.gz  HDP-GPL  HDP-UTILS

```
## 五、配置Ubuntu启动源

```
#以静默模式启动python服务器，你也可以用nginx、httpd方式
root@nm1-machine:/var/www/html# nohup python -m SimpleHTTPServer 1>out.log 2>&1 &

#/etc/apt/sources.list 是包管理工具 apt 所用的记录软件包仓库位置的配置文件，同样的还有位于 /etc/apt/sources.list.d/*.list 的各文件。
root@nm1-machine:/var/www/html# cd /etc/apt/sources.list.d

**因为是Net模式，所以IP可以得改变**
#创建 ambari.list
root@nm1-machine:/etc/apt/sources.list.d# vim ambari.list
#添加以下内容：
deb http://localhost:8000/ambari/ubuntu16/2.7.3.0-139/ Ambari main

#创建 ambari-hdp.list
root@nm1-machine:/etc/apt/sources.list.d# vim ambari-hdp.list
#添加以下内容：
deb http://192.168.159.246:8000/HDP/ubuntu16/3.1.0.0-78/ HDP main
deb http://192.168.159.246:8000/HDP-GPL/ubuntu16/3.1.0.0-78/ HDP-GPL main
deb http://192.168.159.246:8000/HDP-UTILS/ubuntu16/1.1.0.22/ HDP-UTILS main


```

## 六、安装ambari-server
```
#开始安装ambari-server
root@nm1-machine:~# apt-get update
root@nm1-machine:~# apt-get install ambari-server

root@nm1-machine:~# ambari-server setup

#@在选择jdk的时候，选2
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1): 2

#添加JAVA_HOME
Path to JAVA_HOME: /usr/local/software/jdk1.8.0_211
Validating JDK on Ambari Server...done.

#最后启动
root@nm1-machine:~# ambari-server start

#如果报如下错误：
。。。。listener 8080 time lang。。。。
#查看日志
/var/log/ambari-server/ambari-server.log
/var/log/ambari-server/ambari-server.out
#出现如下字样
nm1-machine.com: nm1-machine.com: Name or service not known

#请输入hostname -f，是否存在，如果不存在，请改/etc/hosts
127.0.0.1       localhost
127.0.1.1       nm1-machine.com


#浏览器登录
http://localhost:8080 或者http://nm1-machine.com:8080/#/login
```
## 七、创建集群
```
#看官方例子

#安装到第三步，出现host checks
#警告如右=> ntp or chrony
root@nm2-machine:~# apt install ntp 
root@nm2-machine:~# update-rc.d ntp defaults


```

```
#安装 apt install mysql-server
#设置的root用户、密码为：jiaxin520

#nm1-machine.com、nm2-machine.com（主从服务机都要安装）
root@nm1-machine:~# mysql -u root -p
mysql> CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'hive'@'localhost';
mysql> CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%';
mysql> CREATE USER 'hive'@'nm1-machine.com' IDENTIFIED BY 'hive';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'hive'@'nm1-machine.com';

#创建数据库
mysql> CREATE DATABASE hive;


#安装oozie数据库
root@nm2-machine:~# mysql -u root -p
mysql> CREATE USER 'oozie'@'%' IDENTIFIED BY 'oozie';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'oozie'@'%';
mysql> FLUSH PRIVILEGES;
mysql> CREATE DATABASE oozie;


#安装到Step9
#报错 hive client failed
root@nm1-machine:/var/lib/ambari-server/resources# ln  /usr/share/java/mysql-connector-java.jar /var/lib/ambari-server/resources/mysql-connertor-java.jar
root@nm1-machine:~# ls -l /var/lib/ambari-server/resources/mysql-connertor-java.jar 
#-rw-r--r-- 2 root root 2293132 3月  21 03:56 /var/lib/ambari-server/resources/mysql-connertor-java.jar


```
### 7.1 安装postgresql
 ```
    root@nm1-machine:/usr/share# sudo -u postgres psql

 ```
### 7.2 安装hive数据库
```

root@nm1-machine:/home/nm1# sudo -u postgres psql

postgres=# CREATE DATABASE hive
postgres-# ;
CREATE DATABASE
postgres=# create user hive with password 'hive'
postgres-# ;
CREATE ROLE
postgres=# GRANT ALL PRIVILEGES ON DATABASE hive TO hive;
GRANT


```

### 7.3 安装Oozie
```
root@nm1-machine:/home/nm1# sudo -u postgres psql
postgres=# create database^C
postgres=# CREATE DATABASE oozie;
CREATE DATABASE
postgres=# CREATE USER oozie WITH PASSWORD 'oozie';
CREATE ROLE
postgres=# GRANT ALL PRIVILEGES ON DATABASE oozie TO oozie;
GRANT

```
# 八、删除服务
```
 #因为hadoopCluter部署时，出现500 status code received on DELETE method for API: /api/v1/clusters/HadoopCluster
 #所以就得重新删了
 
 #步骤一  先查询
 root@nm1-machine:~# curl -u admin:admin http://192.168.159.246:8080/api/v1/clusters
 
 


```

# 九、问题
## 9.1 HDFS >  Connection failed: [Errno 111] Connection refused to nm1-machine.com:618
```

```
# 十、出问题看log
```
 #路径为 /var/log
```


============================================

>参考[文档](https://docs.hortonworks.com/HDPDocuments/Ambari-2.7.3.0/bk_ambari-installation/content/ch_Getting_Ready.html)

> 参考 [ambari使用最全面解析](https://zhuanlan.zhihu.com/p/29673506)

> 参考   [ambari cluster not working, error in history server](https://community.hortonworks.com/questions/141235/ambari-cluster-not-working-error-in-history-server.html)

> [错误问题=>186090](https://community.hortonworks.com/questions/186090/mysql-connector-javajar-due-to-http-error-http-err.html)




**特此记录：**
==Ubuntu 18改静态IP  开始==

打开 /etc/netplan/01-network-manager-all.yaml 配置文件，原文内容如下

```
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
```

修改后的配置

```
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager

  ethernets:
    ens33:   #配置的网卡名称
      dhcp4: no    #dhcp4关闭
      dhcp6: no    #dhcp6关闭
      addresses: [192.168.117.130/24]   #设置本机IP及掩码
      gateway4: 192.168.117.2   #设置网关
      nameservers:
          addresses: [114.114.114.114, 8.8.8.8]   #设置DNS
```
==Ubuntu 18改静态IP  结束==

*~~ssh免密登录其他节点服务器~~*

```
#测试Ubuntu是否安装ssh
ssh localhost
#出现了ssh: connect to host localhost port 22: Connection refused
#安装ssh
apt install openssh-server
```

ssh-keygen -t rsa




scp /root/.ssh/id_rsa.pub nm2-machine.com:/tmp/












