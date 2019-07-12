---
title: windows下SkyWalking分布式链路跟踪系统踩坑记录
date: 2018-12-11
tags: [java]
categories: Java
toc: true
---
### 简介
&emsp;&emsp;SkyWalking 是一个针对分布式系统的 APM 系统，也被称为分布式追踪系统。
&emsp;&emsp;skywalaking总体架构分为三部分：

{% blockquote %}
&emsp;&emsp;skywalking-collector：链路数据收集器，数据可以落地ElasticSearch，单机也可以落地H2，但不推荐
&emsp;&emsp;skywalking-web：web可视化平台，用来展示落地的数据
&emsp;&emsp;skywalking-agent：探针，用来收集和发送数据到归集器
{% endblockquote %}

&emsp;&emsp;当然，实际上部署起来其实很简单，不需要了解原理就可以正常使用。

### 部署
#### 安装ElasticSearch 
&emsp;&emsp;必须选择5.X版本的ElasticSearch[下载](https://www.elastic.co/downloads/past-releases),最新的6.X以上的版本暂不兼容(文档没看仔细，在这浪费了不少时间)，同时要求JDK8+
#### 下载SkyWalking安装包
&emsp;&emsp;获取Apache的最新发布的稳定版本或者下载github上面的源码在稳定版本的tag上进行编译，我选择的是5.0的版本[下载](https://github.com/apache/incubator-skywalking/releases)，解压后目录结构如下：
<div align=center>{% asset_img 1.png 版本 %}</div>
<br/>

#### 配置修改
ElasticSearch:
 
 {% codeblock config/elasticsearch.yml   lang:java  %}
 # 修改 
 # 如果 cluster.name 不设置为 CollectorDBCluster ，则需要修改 SkyWalking 的配置文件 
 cluster.name: CollectorDBCluster 
 network.host: 0.0.0.0 
 # 增加 
 thread_pool.bulk.queue_size: 1000
 {% endcodeblock %}
 
 skywalking:
 config/application.yml :
 <div align=center>{% asset_img 2.png 版本 %}</div>
 <br/>
 
#### 运行 
&emsp;&emsp;运行bin 目录下的start.sh/start.bat脚本，会同时启动Collector和Web UI。访问localhost:8080，可以进入web页面，默认账号密码都是admin，具体可以在webapp.yml配置。遇到下面错误可参考解决方法：

{% blockquote %}
服务器500：多半是配置上的问题，可能是collector未正常连接，先去logs目录下查看对应日志；

找不到Collector：检查下webapp/webapp.yml  中collector.ribbon.listOfServers 配置的地址是否正确;

找不到9200端口：可能是config/application.yml中ElasticSearch节点配置不正确，Elastic启动时会启动两个端口，9300是tcp通讯端口，集群间和TCPClient都走的它，9200是http协议的RESTful接口，这里应该配置9300端口.
{% endblockquote %}

#### Agent探针
&emsp;&emsp;拷贝apache-skywalking-apm-incubating目录下的agent目录到应用程序位置，探针包含整个目录，不要改变目录结构；java程序启动时，增加JVM启动参数，-javaagent:/path/to/agent/skywalking-agent.jar。参数值为skywalking-agent.jar的绝对路径
   <div align=center>{% asset_img 3.png 版本 %}</div>
   <br/>
   
&emsp;&emsp;同时修改agent.config配置中的agent.application_code为要监控的应用的名字，其他默认即可。
    <div align=center>{% asset_img 4.png 版本 %}</div>
     <br/>
  
  
&emsp;&emsp;最后启动应用
   <div align=center>{% asset_img 5.png 版本 %}</div>
    <br/>

### elasticsearch-head
  &emsp;&emsp;elasticsearch-head是一个ElasticSearch的可视化监控插件，可以用来监视ElasticSearch运行情况，安装起来很简单，但是需要安装Node环境。
  
   {% codeblock %}
  git clone git://github.com/mobz/elasticsearch-head.git
  cd elasticsearch-head
  npm install
  npm run start
   {% endcodeblock %}
   
  &emsp;&emsp;然后打开 http://localhost:9100/ 就能看到ElasticSearch节点信息了
  <div align=center>{% asset_img 6.png 版本 %}</div>
       <br/>