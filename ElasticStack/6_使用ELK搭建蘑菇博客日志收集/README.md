# 用 ELK 收集多模块站点日志

## 前言

前几篇分别写了 Elasticsearch、Beats、Kibana 和 Logstash。这一篇把它们用到一个分成多个进程的内容站点上：站点前台、管理后台、通知服务、图片服务，以及入口处的反向代理。每个进程写自己的日志文件，Filebeat 在应用机上采集，Logstash 按来源写入不同索引，Kibana 只负责查看。

组件的安装细节不再重复，需要时回到：

- [Elasticsearch 介绍与安装](../1_ElasticSearch介绍与安装/README.md)
- [Beats 入门](../2_Beats入门简介/README.md)
- [Kibana 安装与界面](../3_Kibana安装与介绍/README.md)
- [Logstash 入门](../4_Logstash入门简介/README.md)
- [综合案例](../5_ElasticStack综合案例/README.md)

## 准备运行环境

不依赖别人打包好的整机镜像。实验可以用官方发行包装在同一台 Linux 上，也可以用官方镜像各起一个容器，端口保持下面这一组，后面的配置就能对上：

| 端口 | 用途 |
| ---- | ---- |
| 9200 | Elasticsearch HTTP |
| 5601 | Kibana |
| 5044 | Logstash 的 Beats 输入 |
| 9600 | Logstash 监控 API |

容器方式只映射需要的端口，不要为了图省事打开特权模式。SSH 进容器不是这条管道的必要步骤，配置文件用挂载卷放在宿主机上更不容易随着容器删除一起丢掉。

镜像拉好或包解压好之后，目录里应能看到 Elasticsearch、自带或系统已有的 JDK、Kibana 和 Logstash。Filebeat 不要装在这台 ELK 机器上，它跟应用进程放在一起。

## 启动 Elasticsearch

Elasticsearch 用 mmap 管理索引，系统默认的虚拟内存映射数量往往不够。在宿主机（不是容器内的临时 shell）调整：

```bash
# /etc/sysctl.conf
vm.max_map_count=262144
```

```bash
sysctl -p
```

然后切换到专用账号启动。root 会被拒绝：

```bash
su - observe
cd /opt/observe/elasticsearch
./bin/elasticsearch -d
```

`curl http://127.0.0.1:9200` 返回带版本号和集群名的 JSON，说明 HTTP 层已经就绪。浏览器访问 `http://<ELK主机>:9200/` 看到同样的 JSON 即可，不必安装额外的可视化插件。

## 启动 Logstash

Logstash 接收 Filebeat 的事件，按 `origin` 字段选择索引，再写入 Elasticsearch。配置文件 `site-logs.conf`：

```ruby
input {
  beats { port => 5044 }
}
filter {
  mutate {
    split => { "message" => "|" }
  }
  mutate {
    add_field => {
      "memberId" => "%{[message][1]}"
      "action"   => "%{[message][2]}"
      "actedAt"  => "%{[message][3]}"
    }
  }
  mutate {
    convert => {
      "memberId" => "integer"
      "action"   => "string"
      "actedAt"  => "string"
    }
  }
  mutate {
    remove_field => ["host"]
  }
}
output {
  if [origin] == "site-web" {
    elasticsearch {
      hosts => ["http://127.0.0.1:9200"]
      index => "site-web-%{+YYYY.MM.dd}"
    }
  }
  if [origin] == "site-admin" {
    elasticsearch {
      hosts => ["http://127.0.0.1:9200"]
      index => "site-admin-%{+YYYY.MM.dd}"
    }
  }
  if [origin] == "site-notice" {
    elasticsearch {
      hosts => ["http://127.0.0.1:9200"]
      index => "site-notice-%{+YYYY.MM.dd}"
    }
  }
  if [origin] == "site-media" {
    elasticsearch {
      hosts => ["http://127.0.0.1:9200"]
      index => "site-media-%{+YYYY.MM.dd}"
    }
  }
  if [origin] == "site-edge" {
    elasticsearch {
      hosts => ["http://127.0.0.1:9200"]
      index => "site-edge-%{+YYYY.MM.dd}"
    }
  }
}
```

`origin` 不是日志正文里解析出来的，而是 Filebeat 在采集时打上的标签，并且提升到事件根上。Logstash 用它做路由，每个模块一天一个索引，避免五个模块的字段互相污染映射。没有任何一个 `if` 命中时，事件会被悄悄丢掉，所以新增模块时要同时改采集配置和这里的分支。

```bash
# 前台，便于看启动日志
./bin/logstash -f ./site-logs.conf

# 确认无误后再放后台
nohup ./bin/logstash -f ./site-logs.conf > /var/log/logstash/site.out 2>&1 &
```

管道编译完成前，5044 还不会接受连接。监控端口 9600 有响应，只说明进程活着，不代表 Beats 输入已经就绪。看日志里的 pipeline started 再启动 Filebeat。

## 启动 Beats

### 启动 Filebeat

Filebeat 是轻量的文件采集器，和写日志的应用放在同一台机器。ELK 那台机器上不启动它。若应用机还没装，按 Beats 那一篇的步骤解压一份，版本与 Elasticsearch 对齐。

配置 `site-logs.yml`，每个输入盯一个模块的日志，并用 `fields` 写上来源。`fields_under_root: true` 让 `origin` 出现在事件顶层，Logstash 的 `[origin]` 才能匹配到。

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /srv/site/web/logs/app.out
    fields:
      origin: site-web
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /srv/site/admin/logs/app.out
    fields:
      origin: site-admin
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /srv/site/notice/logs/app.out
    fields:
      origin: site-notice
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /srv/site/media/logs/app.out
    fields:
      origin: site-media
    fields_under_root: true

setup.template.settings:
  index.number_of_shards: 1
output.logstash:
  hosts: ["10.0.4.20:5044"]
```

`10.0.4.20` 换成 ELK 机器的地址。反向代理的访问日志若也要进同一条管道，再加一个输入，`origin` 设为 `site-edge`，路径指向代理的 access 日志。

```bash
./filebeat -e -c site-logs.yml
```

后台运行：

```bash
#!/bin/bash
nohup ./filebeat -e -c site-logs.yml > /var/log/filebeat/site.out 2>&1 &
```

启动日志里应列出已经打开的文件。若 harvester 数为 0，先检查路径和权限，Filebeat 的运行用户要能读这些日志。

## 启动 Kibana

回到 ELK 机器，仍用非 root 账号：

```bash
su - observe
cd /opt/observe/kibana
./bin/kibana
```

监听 5601 后，浏览器访问 `http://<ELK主机>:5601`。为每个前缀建索引模式，例如 `site-web-*`、`site-admin-*`，时间字段选 `@timestamp`。Discover 里按 `origin` 筛选，可以确认路由没有串模块。

仪表盘上的柱状图、饼图做法与综合案例相同，把索引模式换成对应模块即可。这里不再逐步贴界面。索引模式建错、或时间范围太窄，是这一步最常见的“日志明明已经入库却什么都看不到”。
