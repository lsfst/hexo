---
title: windows idea 搭建Node开发环境
date: 2018-11-09 11:35:58
tags: [node]
categories: node
---

最近有点小闲，开始学习NodeJS相关的东西。之前Node与NPM已经安装好了，环境变量也设置ok：

{% asset_img node1.png 版本 %}

话不多说，npm install 四连走起：

{% codeblock  lang:bash  %}
npm install express -g 
npm install jade -g
npm install mysql -g
npm install -g express-generator
{% endcodeblock %}

在安装jade时直接报警告了:

{% asset_img node2.png jade警告 %}

原来是改名成pug了，问题不大，继续下一步！

在控制台运行：
{% asset_img node3.png 一切正常 %}

由于平时干后端习惯了使用idea，而且idea也提供了NodeJS插件，所以接下来直接在idea中安装NodeJS插件。
打开idea，File -> setting ->Plugins,右边默认没有这个组件，需要手动点击Browe repositories..，在插件列表中搜索nodejs,将看到NodeJS插件，点击下载：

{% asset_img node4.png 下载 %}

然后：

{% asset_img node5.png 下载失败 %}

不慌，可以直接去 [idea plugin 官网](https://plugins.jetbrains.com/)下载插件
直接输入NodeJS，回车，插件版本有很多，选了一个最新的版本下载，解压，将解压文件夹里面的NodeJS子文件夹复制到idea所在安装目录的plugins子文件夹中

{% asset_img node6.png 解压复制 %}

之后重启idea，安装按钮消失，看起来是ok了

{% asset_img node7.png 按钮消失 %}

File -> new ->Project创建应用：

{% asset_img node8.png 创建 %}

{% asset_img emoji1.png emmmmm %}

没事，还有一个方法，将之前放在plugins目录下的NodeJS文件夹删除，将压缩包直接放在plugins目录下，idea可以直接解压插件压缩包：

{% asset_img node9.png 压缩包 %}

结果还是不行：

{% asset_img node10.png 版本不兼容 %}

原来是插件与idea的版本不兼容，需要适配。

查看idea版本：Help-->about

{% asset_img node11.png idea版本 %}

这个版本对应着插件的COMPATIBILITY，只要idea版本介于插件的COMPATIBILITY范围内即可。

{% asset_img node12.png idea版本 %}

符合要求的版本有几个，随机选择一个下载：

{% asset_img node13.png 插件版本 %}

直接Install plugin from disk ，加载压缩包，重启idea：

{% asset_img node14.png 重启 %}

终于出来了

然后创建第一个应用：发现红框处无法获取版本，莫不是加载方式的锅？

{% asset_img node15.png 重新加载 %}

于是又尝试着直接将压缩包解压的文件夹放在plugins目录下，这次可以正常获取版本了

最后总结一下这个插件安装流程：
1.File -> setting ->Plugins->Browe repositories，搜索nodejs,下载
2.若下载失败，去 [官网](https://plugins.jetbrains.com/)下载对应插件压缩包，将压缩包解压，NodeJS文件夹直接放在idea安装目录下的plugins子目录下
3.重启idea，就可以正常创建

一定要注意的点：
1.插件与idea版本的匹配
2.Install plugin from disk加载压缩包在创建应用时无法正常获取express版本，最好直接解压放在idea安装目录的plugins下