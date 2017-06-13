---
title: elk简单环境搭建 for linux
date: 2017-06-13 19:39:30
tags: [elk]
---
环境：ubuntu 16.06虚拟机：4核8G内存
在官网下载的`tag.gz`包，官网地址[https://www.elastic.co/webinars/introduction-elk-stack](https://www.elastic.co/webinars/introduction-elk-stack)
安装版本是**5.4.1**，本文只安装了`Elasticsearch`、`Logstash`、`Kibana`
<!--more-->
#### Elasticsearch
1. 下载压缩包并解压
2. 在es的根目录下`config/elasticsearch.yml`文件，内容如下
``` yml
# Use a descriptive name for the node:
node.name: xuannode  ##不要有'-'、'_'、'+'
# Path to directory where to store the data (separate multiple locations by comma):
#
path.data: /home/huangyuan/elk/elasticsearch/data
#
# Path to log files:
#
path.logs: /home/huangyuan/elk/elasticsearch/logs/*
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 192.168.1.179
#
# Set a custom port for HTTP:
#
http.port: 9200
discovery.zen.ping.unicast.hosts: ["192.168.1.179"]
http.cors.enabled: true
http.cors.allow-origin: "*"
```
#### logstash
1. 下载压缩包并解压
2. 创建一个`logstash.conf`文件，输入以下内容并保存:
```conf
input{
	file {
		path => "/home/huangyuan/elkdata/*.log"
	}
}
output {
	elasticsearch {
		hosts => "http://192.168.1.179:9200"
		index => "logstash-%{+YYYY.MM.dd}"
	}
    stdout {}
}
```
3. 启动时执行 `bin/logstash -f logstash.conf`
#### kibana
1. 下载压缩包并解压缩
2. 编辑`config/kibana.yml`
``` yml
# Kibana is served by a back end server. This setting specifies the port to use.
server.port: 5601
# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "192.168.1.179"
# The Kibana server's name.  This is used for display purposes.
server.name: "xuankibina"
# The URL of the Elasticsearch instance to use for all your queries.
elasticsearch.url: "http://192.168.1.179:9200"
```
----
启动的时候依次启动 es、logstash、kibana就可以了
PS:在LogStash的配置文件`logstash.conf`中,`input`配置的就是logstash要监听的文件路径，启动之后，先在监听的文件夹中创建一个log文件并输入随意内容。
<hr>
以上