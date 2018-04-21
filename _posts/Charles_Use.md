---
title: '使用Charles对Android App进行抓包'
date: 2017-02-23 23:43:40
tags: [Charles]
categories: [Android,other]
---
## 使用Charles对Android App进行抓包

记录一下，方便日后查找

### 1.软件下载地址
这里就不提供啦。

### 2.Mac配置
1.Help --> SSL Proxying --> Install Charles Root Certificate
2.Proxy Settings --> port 8888 Enable transparent HTTP proxying
3.SSL Proxying Settings --> Enable SSL Proxying 同时添加需要监听的接口，可使用*通配符

### 3.Android手机配置
1.连接和Mac的同一个WIFI，同时设置此WIFI的代理服务器，ip即这台电脑连接的WIFI的ip，端口8888.
2.Help --> SSL Proxying --> Save Charles Root Certificate 然后把证书拷贝到手机上面安装