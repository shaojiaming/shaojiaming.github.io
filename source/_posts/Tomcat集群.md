---
title: Tomcat集群（linux） 
date: 2017-03-17 15:01:49
categories: 力量之源
tags: [java,tomcat,apache]
---
本文在red hat下进行搭建的，参考网上的搭建步骤结合自己的实际操作，从系统安装到最终的搭建进行详细的说明。
<!--more-->
## 目录
	一.	前期准备	
	二.	环境安装	
		1.	安装gcc 和g++	
		2.	安装apache	
		3.	配置java运行环境变量	
		4.	配置tomcat	
		5.	配置JK连接器	
	三.	测试apache负载均衡是否生效	


## 前期准备
### 安装介质 :

|工具   | 说明 |
| ------------------------------- | ------------------------------------- |
| rhel-server-6.4-x86_64-dvd.iso  | RedHat Enterprise Linux 6.4 x86 64bit |
| tomcat-connectors-1.2.41.tar.gz | Tomcat负载均衡模块                    |
| httpd-2.4.10.tar.gz             | Apache源码包                          |
| apr-1.5.1.tar.gz                | Apache依赖包                          |
| apr-util-1.5.1.tar.gz           | Apache依赖包                          |
| pcre-8.32.tar.gz                | Apache依赖包                          |
| Tomcat 7.0.75                      | Tomcat安装包                          |
| jdk-7u79-linux-x64.tar.gz       | Java运行环境                          |

## 环境安装
###	关闭linux selinux
1、	root用户登录
2、	cd /etc/selinux
3、	vim config   设置selinux=disabled
Selinux 解释：
SELinux是一种基于域-类型 模型（domain-type）的强制访问控制（MAC）安全系统，它由NSA编写并设计成内核模块包含到内核中。

### 关闭防火墙
1、	运行 service iptables status  查看防火墙状态
2、	如果提示防火墙未关闭请执行以下操作
3、	chkconfig iptables off  (关闭防火墙-永久生效)
4、	chkconfig iptables on  (打开防火墙-永久生效)
5、	重启服务器 reboot

###	安装gcc 和g++
挂载光驱

``` stylus
# mkdir /mnt/cdrom
# mount -o loop -t iso9660 /opt/software/rhel-server-6.4-x86_64-dvd.iso /mnt/cdrom/
# rpm -ivh /mnt/cdrom/Packages/yum-3.2.29-40.el6.noarch.rpm
# vi /etc/yum.repos.d/local.repo
[rhel-Server]
name=Server
baseurl=file:///mnt/cdrom/Server
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
# cat /etc/yum.repos.d/local.repo
# yum clean all
# yum list gcc*
# yum install gcc.x86_64
# yum install gcc-c++.x86_64
```
验证
 !["图一"](/images/1489737763633.png)

###	安装apache(官网：http://httpd.apache.org/)

``` stylus

中间件存储路径/usr/local/
Apache安装路径/usr/local/web/apache
#tar -zxvf httpd-2.4.10.tar.gz
#tar -zxvf pcre-8.32.tar.gz
#tar -zxvf apr-1.5.1.tar.gz
#tar -zxvf apr-util-1.5.1.tar.gz

#cd pcre-8.32
#./configure --prefix=/usr/local/pcre
#make
#make install

#cd ../apr-1.5.1
#./configure --prefix=/usr/local/apr
#make
#make install

#cd ../apr-util-1.5.1
#./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr
#make
#make install

#cd ../httpd-2.4.10
#./configure --prefix=/usr/local/web/apache --enable-so --enable-mods-shared=most --with-mpm=worker --enable-proxy --enable-rewrite --with-pcre=/usr/local/pcre --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util
#make
#make install

启动apache
#/usr/local/web/apache/apachectl start

```
在本地打开浏览器，访问http://127.0.0.1
如果出现“It Works!”则表示启动成功了
 
 

###	配置java运行环境变量

``` stylus
#mkdir /usr/java
#tar –zxvf  jdk-7u79-linux-x64.tar.gz  /usr/java
#vi /etc/profile
export JAVA_HOME=/usr/java/jdk1.7.0_05
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
export JAVA_HOME CLASSPATH PATH
#source /etc/profile
```
验证:输入 
javac
java –version
 
###	配置tomcat
1.将tomcat分别放到两个文件夹下
2.修改service.xml文件的port
3.修改两台server的端口 
4.添加jvmRoute(配置另一台时jvmRoute=”s2”)
`<Engine defaultHost="localhost" name="Catalina" jvmRoute="tomcat1">`
5.解开注释
`<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>`
6.启动两台tomcat 看是否能访问到对应的测试页面

###	配置JK连接器
1.编译jk

``` stylus
# cd tomcat-connectors-1.2.32-src 
# cd native/
# ./configure --with-apxs=/usr/local/web/apache/bin/apxs 
# make  
# ls
# cd apache-2.0/
# ls 
bldjk54.qclsrc  Makefile.apxs     mod_jk.a    mod_jk.lo
bldjk.qclsrc    Makefile.apxs.in  mod_jk.c    mod_jk.o
config.m4       Makefile.in       mod_jk.dsp  mod_jk.so
Makefile        Makefile.vc       mod_jk.la   NWGNUmakefile
# sudo cp ./mod_jk.so /usr/local/web/apache/modules/
```

2.修改apache的httpd.conf文件
添加

``` stylus
LoadModule jk_module modules/mod_jk.so
<IfModule jk_module>
  JkWorkersFile conf/workers.properties
  JkLogFile logs/mod_jk.log
  JkLogLevel warn
  JkMount /*.jsp loadBalanceServers
</IfModule>
```
3.在conf文件夹下面添加workers.properties文件
``` stylus
--------------------------------------------------------------------------------------------------------------------------
#
# workers.properties
#
# list the workers by name
worker.list=loadBalanceServers
# localhost server 1
# ------------------------
worker.s1.port=6009 //6080端口转发port
worker.s1.host=localhost  #tomcat的主机地址，如不为本机，请填写ip地址
worker.s1.type=ajp13   #ajp13 端口号，在tomcat下server.xml配置,默认8009
worker.s1.lbfactor=10   #server的加权比重，值越高，分得的请求越多
# localhost server 2
# ------------------------
worker.s2.port=7009 //7080端口转发port
worker.s2.host=localhost  #tomcat的主机地址，如不为本机，请填写ip地址
worker.s2.type=ajp13  #ajp13 端口号，在tomcat下server.xml配置,默认8009
worker.s2.lbfactor=10  #server的加权比重，值越高，分得的请求越多

worker.loadBalanceServers.type=lb
worker.loadBalanceServers.balanced_workers=s1,s2  #  #controller控制的tomcat的名称指定分担请求的tomcat由tomcat中的server.xml中设值
worker.loadBalanceServers.sticky_session=true  #会话是否有粘性，false表示无粘性，同一个会话的请求会到不同的tomcat中处理

# worker.controller.retries=3  #请求失败以后重试次数
# worker.controller.sticky_session_force=false #当一个节点崩了，如果设置为true，那么服务器返回500错误给客户端，如果设置为false,则转发给其他的tomcat，但是会丢失回话信息
```

4.重启apache

## 测试apache负载均衡是否生效
1.	创建两个项目test
2.	在index.jsp中分别添加如下代码
``` stylus
<%session.setAttribute("id","123");%>
<%=session.getAttribute("id")%>
```
分别重启两台tomcat服务器, 在两台浏览器中输入http://10.46.2.10/test/index.jsp


