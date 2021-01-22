---
title: 一次 ElasticSearch 宕机事故处理
date: 2017-07-21 14:40:34
description: 初次接触使用 ElasticSearch，发现服务进程频繁宕机，排查后发现是堆内存不足导致
tags: ["技术"]
---

早上，当我连上服务器查看昨晚临走时开的数据导入 es 的任务完成度，发现完成到 78% 时挂了

访问 Kibana 查看 es 状况，red 警报，`ps aux | grep elastic` 查看 es 服务的进程，没有相关进程

执行 `systemctl status elasticsearch`，显示 failed
执行 `systemctl start elasticsearch` 重新启动 es 服务，启动成功

<!--more-->

```bash
http :9200/_cat/indices
```

查看索引，没问题

```bash
http :9200/userinfo/myspace/_count
```

查看某个type的数量，响应很慢，返回error信息，es服务再次挂了
重新启动服务，再次查看，又挂了，这样来回多次，我觉得应该es服务出了问题，但是一头雾水，不知缘故

```json
机器配置：
 24核
 48G内存
 1.3T硬盘
```

截止昨晚，es 里导入了 8 亿多条数据，占用 305G 左右的存储，没有出错

Google 了很久，找到下面两篇文章，大概了解问题原因
> Elasticsearch配置及优化  http://news.deepaso.com/search-engine/elasticsearch-config-performance.html
> 官方文档 - 堆内存：大小和交换 https://www.elastic.co/guide/cn/elasticsearch/guide/current/heap-sizing.html

Elasticsearch 默认安装后设置的堆内存是 1 GB，这是 es 屡次宕机的原因，我把它设置为机器内存的一半，`export ES_HEAP_SIZE=22g`，重启 es 服务发现没有生效，可能因为我是通过 Ubuntu 的包管理器 `apt` 安装的 ElasticSearch，然后用的 Ubuntu 系统工具 `systemctl` 控制的 es 的服务进程，这种情况下 es 启动时不会从当前 shell 读取 `export` 设置环境变量

换种方式：

```bash
sudo vi /usr/share/elasticsearch/bin/elasticsearch
```

修改 es 启动脚本文件，增加一行 `ES_JAVA_OPTS="-Xms22g -Xmx22g"`
这样等于执行 `./bin/elasticsearch -Xmx22g -Xms22g` 来启动 es 服务进程
具体es的启动配置选项可以看启动脚(`/usr/share/elasticsearch/bin/elasticsearch`)
改完配置重新启动 es，`htop` 查看内存情况，很明显内存占用上来了，没改前是 3g+ 的内存占用，现在是 20 多 g
![](https://github.com/Zhiwei1996/Zhiwei1996.github.io/raw/master/images/elasticsearch-server-status.png)
一切恢复正常 :)