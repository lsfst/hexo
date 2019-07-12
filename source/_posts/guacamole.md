---
title: Apache Guacamole 使用记录
date: 2019-03-17
tags: [java]
categories: Java
toc: true
---
#### 简介
&emsp;&emsp;Guacamole是由多个模块组成的无客户端的远程桌面网关，它支持VNC,RDP,SSH等标准协议。guacamole包括两大部分，guacamole-client和guacamole-server。client是一个web服务器，实现了对server的远程访问。server则实现了client和远程桌面服务的桥梁。用户通过浏览器连接到Guacamole的服务端。Guacamole的客户端是用JavaScript编写的，Guacamole server通过web容器（比如tomcat）把服务提供给用户。一旦加载，客户端通过http承载着Guacamole自己的定义的协议与服务端通信。部署在Guacamole server这边的Web应用程序，解析到Guacamole protocal，就传给Guacamole的代理guacd（中间层），这个代理替用户连接到远程机器。
           
#### 安装流程
[下载地址](http://guacamole.apache.org/releases/0.9.14/)
[官方文档](http://guacamole.apache.org/doc/gug/index.html)

操作系统:CENTSOS 6.9+tomcat8+jdk8
客户端可以直接下载war包在tomcat使用，但服务端必须要下载源码在本地编译.

没时间解释了，具体流程参考[文档](http://guacamole.apache.org/doc/gug/installing-guacamole.html)

tomcat安装
{% blockquote %}
yum install tomcat
{% endblockquote %}

安装GCC编译环境
{% blockquote %}
yum install gcc
{% endblockquote %}

安装依赖
{% blockquote %}
yum -y install cairo-devel libjpeg-devel libpng-devel uuid-devel 
yum -y install ffmpeg-devel freerdp-devel pango-devel libssh2-devel 
yum -y install libtelnet-devel libvncserver-devel pulseaudio-libs-devel 
yum -y install openssl-devel libvorbis-devel libwebp-devel 
yum -y install freerdp-plugins
{% endblockquote %}

下载服务端压缩包
{% blockquote %}
tar -xzvf guacamole-server-0.9.14.tar.gz
{% endblockquote %}
进入解压目录
{% blockquote %}
cd /guacamole-server-0.9.14/
{% endblockquote %}
编译,如果检测状态含有no，需要检查上方的第三方库是否安装正确。
{% blockquote %}
./configure --with-init-dir=/etc/init.d
{% endblockquote %}
安装
{% blockquote %}
make
make install
{% endblockquote %}
启动guacd服务
{% blockquote %}
/etc/init.d/guacd start
{% endblockquote %}
客户端直接将war包放在/var/lib/tomcat/webapps目录下,重启tomcat
{% blockquote %}
/etc/init.d/tomcat restart
{% endblockquote %}
访问http://ip:8080/guacamole
 {% asset_img 1.png %}
 这时候能看到页面，但还不能登陆，还需要配置
 
 默认情况下，Guacamole的默认配置目录在linux下是/etc/guacamole，在windows下则是C:\Users\curentUser\.guacamole,需要在对应位置手动创建.guacamole这个目录
 在该目录下新建guacamole.properties，这是主要的Guacamole配置文件。该文件中的属性规定了Guacamole将如何连接到guacd。在文件中写入：
{% codeblock %}
 guacd-hostname:localhost
 guacd-port:4822
 {% endcodeblock %}
 新建user-mapping.xml  写入
 {% codeblock %}
 <user-mapping>
     <authorize username="test" password="test">
         <protocol>vnc</protocol>
         <param name="hostname">192.168.162.15</param>
         <param name="port">5900</param>
     </authorize>
 </user-mapping>
   {% endcodeblock %}
 这里是说，新增一个vnc服务器，服务器地址为192.168.162.15，vnc所有端口为5900，guacamole连接这个vnc服务器的用户名和密码为test，test。如果想配置多个连接，就写多个authorize节点。
 
 最后在服务器上搭建好VNC服务就可以在客户端进行访问了。
