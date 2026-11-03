# Elasticsearch 安装与检索

## Elastic Stack 简介

ELK 是 Elasticsearch、Logstash、Kibana 三个组件的旧称。采集器 Beats 加入后，整套更常叫做 Elastic Stack。名字换了，分工没有变：采集、加工、存储检索、展示。

### Elasticsearch

Elasticsearch 用 Java 编写，底层索引是 Lucene。它把索引分成片，可以在多台机器上保存副本，对外提供 REST 接口。集群能自动发现节点，查询可以落在不同分片上并行执行。它适合全文检索、日志和指标一类“先写入、再按条件查”的数据，不是用来替代事务数据库的。

### Logstash

Logstash 同样基于 Java，是一条可配置的管道：从多种输入读事件，解析、改字段，再写到存储。早期没有 Beats 时，它也承担采集。现在更常见的是让它只做加工，因为每台机器都跑一个 JVM 成本偏高。

### Kibana

Kibana 基于 Node.js，给 Elasticsearch 提供查询和图表界面。它不单独保存业务文档，仪表盘上的数字都来自索引里的检索。

### Beats

Beats 是装在数据源旁边的轻量采集器，可以直接写 Elasticsearch，也可以先交给 Logstash。常见成员：

- Packetbeat：抓取主机之间的流量，解析 HTTP、DNS、MySQL、Redis 等应用协议，把请求和响应关联成事件
- Filebeat：转发日志文件，用来代替早期的 logstash-forwarder
- Metricbeat：按周期拉取系统和应用的指标，对象包括 Nginx、MySQL、Redis、系统本身等

采集用 Beats，需要切割字段时再进 Logstash，是目前比较稳妥的分工。

## Elasticsearch 快速入门

### 简介

Elasticsearch 是在 Lucene 之上的分布式搜索与分析引擎，HTTP 接口使用 JSON。它的设计目标是：从一台机器开始，同一套 API 可以扩到多台；索引和查询近实时；多租户时用索引把数据隔开。给一个已有的应用加搜索，不必自己管理段合并、倒排和分片。

在 Stack 里它处于中心位置：Beats 和 Logstash 把数据送进来，Kibana 从这里读出去。

### 版本

5.0 之前，ELK 各组件的版本号互不相干，混装很容易在模板和映射上不兼容。从 5.0 起统一版本号。本篇的安装、映射和 Java 客户端按 7.x 来写。7.x 默认不再使用自定义文档类型，类型名固定为 `_doc`。笔记里若出现 `include_type_name`，只是为了对照旧教程，新索引不必再造类型名。

### 获取发行包

从 Elastic 的下载页选择与操作系统匹配的 7.x 包，Linux 实验用 `linux-x86_64` 的 tar。不要混用 6.x 的配置键。校验压缩包的校验和之后再解压，避免装到残缺文件上。

### 用容器提供一台 Linux

如果本机不是 Linux，可以起一个带 SSH 的系统容器，把 9200 和 9300 映射出来。9200 是 REST，9300 是节点之间的传输口。容器镜像用发行版官方镜像即可，不必使用他人打包的整套环境。

```bash
docker run -d --name es-lab \
  -p 2222:22 -p 9200:9200 -p 9300:9300 \
  -v /etc/localtime:/etc/localtime:ro \
  centos:7 /usr/sbin/init
```

实验机若已经是 Linux，跳过这一步，直接在宿主机安装。容器里跑 Elasticsearch 还要处理内存锁定和文件描述符，单机学习更省事的做法是在虚拟机里按下面的步骤装。

### 单机安装

Elasticsearch 拒绝以 root 启动。先建专用账号和目录：

```bash
useradd observe
mkdir -p /opt/observe
tar -xzf elasticsearch-7.17.9-linux-x86_64.tar.gz -C /opt/observe
mv /opt/observe/elasticsearch-7.17.9 /opt/observe/elasticsearch
chown -R observe:observe /opt/observe
```

编辑 `config/elasticsearch.yml`。一旦 `network.host` 不是回环地址，进程会按生产模式做一系列检查，内存和文件描述符不够就会拒绝启动。

```yaml
cluster.name: campus-lab
node.name: campus-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
```

单节点实验写上 `discovery.type: single-node`，可以不配置种子主机。这只适合一台进程的练习，不能原样带进多机集群。

堆内存不要超过机器物理内存的一半，还要给操作系统留页面缓存。小虚拟机可以先改 `config/jvm.options`：

```text
-Xms512m
-Xmx512m
```

两个值保持相等，避免运行中再向系统申请堆。

内核参数在宿主机上改，限制一个进程能创建的内存映射数量：

```bash
# /etc/sysctl.conf
vm.max_map_count=262144
sysctl -p
```

### 启动

```bash
su - observe
cd /opt/observe/elasticsearch
./bin/elasticsearch -d
```

`-d` 放进后台。前台启动时日志直接打在终端上，排错更方便。就绪后访问 `http://<主机>:9200/`。返回 JSON 里应有 `cluster_name`、版本号，以及 `tagline`。看不到 JSON 时，先看 `logs/campus-lab.log`，不要反复换端口。

## 错误分析

### 不能用 root 运行

日志出现 `can not run elasticsearch as root` 时，进程是被 root 拉起的。切换到 `observe` 再启动。前面若还有 ML 控制器因为管道写失败而退出，那是子进程跟着父进程一起结束，根因仍是启动账号不对。

### 文件描述符太少

`max file descriptors [4096] ... too low, increase to at least [65536]`。用 root 编辑 `/etc/security/limits.conf`：

```text
observe soft nofile 65536
observe hard nofile 131072
observe soft nproc  4096
observe hard nproc  4096
```

重新登录该用户后，`ulimit -n` 应显示新值。只改文件不重新登录，当前 shell 仍是旧限制。

### 线程数太少

`max number of threads ... is too low, increase to at least [4096]`。在 `/etc/security/limits.d/` 里找到限制 nproc 的文件，把该用户的软限制提到 4096。有的系统把默认值写死为 1024，会盖过 `limits.conf` 里的设置，所以两处都要看。

### 系统调用过滤失败

`system call filters failed to install` 多见于内核不支持 seccomp 的旧系统。Elasticsearch 5.2 之后默认打开 bootstrap 检查。临时做法是在 `elasticsearch.yml` 里关闭：

```yaml
bootstrap.system_call_filter: false
```

这是放弃一项保护，只在确实无法升级内核的实验机上使用。`bootstrap.memory_lock` 是另一项，用来锁定堆、避免交换，和 seccomp 无关，不要为了这一条错误顺便写成 false。

### keystore 权限不对

`AccessDeniedException` 指向 `config/elasticsearch.keystore` 时，文件属主仍是 root。解压或初始化密钥库时若用了 root，专用账号就打不开它。

```bash
chown observe:observe /opt/observe/elasticsearch/config/elasticsearch.keystore
```

整个 `config` 目录的属主最好一并改成 `observe`，避免下一个文件再报同样的错。

### 生产发现配置不完整

`the default discovery settings are unsuitable for production use`，并要求配置 `discovery.seed_hosts`、`discovery.seed_providers` 或 `cluster.initial_master_nodes` 之一。单机实验用上一节的 `discovery.type: single-node`。若坚持不写 single-node，就显式给出初始主节点：

```yaml
node.name: campus-1
cluster.initial_master_nodes: ["campus-1"]
```

`initial_master_nodes` 只在集群第一次选举时使用，节点名必须和 `node.name` 一致。

把上面几类失败放在一起看，顺序通常是：先换对运行账号，再放宽描述符和线程，然后补发现配置，最后才动 seccomp。每一条都会让进程以非零状态退出，HTTP 端口不会处于半开。用 `ss -lnt | grep 9200` 确认没有监听之后，再读日志末尾的 `ERROR`，不要只看终端里被截断的第一行。

数据目录 `data/` 里一旦写过集群的节点 id，再把同一目录拷到另一台机器并改 `cluster.name`，节点会拒绝加入，认为数据属于别的集群。实验想推倒重来时，停掉进程，删掉 `data` 下的内容再启动，而不是只改配置文件里的集群名。

## 可视化：elasticsearch-head

官方发行包不带管理界面。elasticsearch-head 是一个独立的前端，能看索引、分片和做简单查询。可选装法：

- 源码方式用 npm 启动，依赖多，不建议作为第一选择
- Docker 镜像，适合本机没有浏览器插件商店时
- 浏览器插件，请求从浏览器直接发到集群，少一次跨域
- 作为 Elasticsearch 插件安装，旧版本才常见，7.x 不推荐

### 容器方式

```bash
docker run -d --name es-head -p 9100:9100 mobz/elasticsearch-head:5
```

浏览器打开 `http://127.0.0.1:9100`，把集群地址填成 Elasticsearch 的 HTTP 地址。前端和集群不同源，浏览器会拦截请求，除非集群允许跨域：

```yaml
http.cors.enabled: true
http.cors.allow-origin: "http://127.0.0.1:9100"
```

`allow-origin` 尽量写成 head 所在的具体源，而不是星号。改完要重启 Elasticsearch。星号加上带凭证的请求，会把集群接口暴露给任意网页。

### 浏览器插件

插件从浏览器所在的源发起请求，一般碰不到上述跨域限制。装好后同样填写 `http://<主机>:9200`。能连上就可以新建索引、浏览文档。插件只是客户端，集群的安全仍然取决于 9200 谁能访问。实验机不要把 9200 暴露到公网再靠一个前端来“管理”。

## 基本概念

### 索引

索引是一组相关文档的逻辑集合，物理上分成一个或多个分片，每个分片可以有零个或多个副本。它接近关系库里的“库”，而不是一张表：里面可以有结构相近的文档，映射在索引级别定义。7.x 之前常把索引比作库、把 type 比作表；type 废弃之后，一种文档就单独使用一个索引更清晰。

索引为全文检索做了倒排，不按行原样扫描。原始 JSON 仍会以 `_source` 保存，除非你显式关掉。关掉 `_source` 可以省磁盘，但局部更新、重建索引和高亮都会失去原文，实验索引不要关。

按天滚动的日志索引会越来越多。用索引模板把分片数、副本数和公共映射套到匹配的名字上，例如模板匹配 `gate-*`，之后新建的 `gate-2024.06.18` 自动带上这些设置，不用每次 PUT 都写一遍。模板不修改已经存在的索引。旧索引要改设置，只能按索引名单独调用 `_settings`，能改的也只有副本数这类动态项，主分片数仍然改不了。

可组合的索引模板示例：

```http
PUT /_index_template/gate-logs
{
  "index_patterns": ["gate-*"],
  "priority": 50,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" },
        "origin": { "type": "keyword" },
        "status": { "type": "integer" }
      }
    }
  }
}
```

`priority` 用来在多个模板都匹配同一名字时决定谁生效，数字大的覆盖数字小的同名字段。`index_patterns` 用通配符，不要写成已经存在的某一个具体索引，否则下一天的新索引套不上。映射里把 `origin` 定成 keyword，是为了后面按模块做 terms 聚合；若留成动态 text，饼图只能用 `.keyword` 子字段，而有的写入路径不会自动加这个子字段。`status` 用整数，访问日志里的状态码才能做范围过滤。模板写错时，先 `DELETE /_index_template/gate-logs` 再重建，然后删掉已经按错误映射生成的当天索引，让下一条日志重新触发创建。只改模板不删旧索引，旧映射会一直留着。

查看模板是否被某天的索引套用：`GET /gate-2024.06.18/_mapping` 里应能看到 `origin` 为 keyword。若仍是 text，说明索引创建时模板还没写上，或 pattern 没匹配到。用 `GET /_index_template/gate-logs` 回读一次，确认集群接受的内容和编辑器里的一致，避免少了一个逗号却把上一版留在了集群里。回读结果里的 `index_patterns` 和 `priority` 应与提交时相同。若集群上还有旧的模板名，用 `_cat/templates` 把名字列出来，删掉不再使用的那一份，防止两个模板同时匹配 `gate-*` 时字段类型被低优先级的定义盖住。练习结束前记下当前索引的文档数和模板名。下次启动若健康不是绿色，先对照这些名字看是不是旧索引的副本没有节点可放，而不是先去改内核参数。确认无用之后再删索引，删之前看一眼别名，避免把仍被查询使用的物理索引删掉。别名仍指向已删索引时，查询会失败，而模板本身还在，下一条写入会再建出同名索引，容易让人以为删除没有生效。因此每次删除后都再查一次别名列表和索引列表，两份名单里都不再出现练习用的名字，再停止进程。这样下一次实验不会继承上一次留下的分片。若希望保留模板，可以只删索引，下一次启动仍按原模板创建新索引，适合连续做几天的日志实验，也能避免每天的字段类型不一致，查询端就不必跟着改字段名，已经保存的图表也可以继续留用。

生命周期政策可以规定热阶段保留几天、之后删除。没有政策时，练习中的按天索引会一直占磁盘，集群先黄后红，原因看起来像是“突然不能写入”，其实是水位。实验结束删掉 `reader`、`shelf`、`lodging` 和 `gate-*`，再看一次健康颜色，避免下次启动把旧分片算进去。

### 文档

文档是索引里的基本单位，相当于一行记录，用 JSON 表示。同一索引里同一字段的类型必须一致，不能这份文档里 `age` 是数字、下一份又是对象。字段可以重复出现，成为多值字段；也可以是对象或数组，用来表示嵌套结构。

### 映射

映射是字段名到类型、以及该字段如何建索引的规则。文本要不要分词、日期用什么格式、数字用哪一种精度，都写在映射里。它不等于“分析”本身：分析是把文本切成词项的过程，映射里的 `analyzer` 参数决定分析怎么做。

不预先创建映射时，Elasticsearch 会按第一批值猜测类型。猜测一旦写入映射就改不了，所以要检索和聚合的索引应事先写好映射。

### 文档类型

6.x 及更早允许一个索引里放多种 type，例如文章和评论。限制是：不同 type 里同名字段必须同类型。7.x 起一个索引只有一个类型 `_doc`，自定义 type 需要旧参数才勉强打开，8.x 则完全移除。新数据把文章和评论分成两个索引，而不是两种 type。

## REST 接口

集群的管理、写入和查询都走 HTTP 与 JSON。下面用索引 `reader` 演示。实验结束后删掉它，避免和后面的映射示例抢名字。

### 非结构化写入

Lucene 需要知道字段类型。Elasticsearch 允许先不建映射就写入，服务端根据值猜测类型并自动补映射。这对用户是透明的，但猜错之后要重建索引才能改正。

### 创建空索引

```http
PUT /reader
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 0
  }
}
```

单机没有别的节点可放副本时，副本数保持 0，集群才是绿色。分片数在索引创建后不能修改。

### 删除索引

```http
DELETE /reader
```

确认信息里 `acknowledged` 为 true 表示集群已接受删除。这个操作删的是整个索引，不是一条文档。

### 插入文档

7.x 的路径是 `POST /{索引}/_doc/{id}`，不必再写类型名。

```http
POST /reader/_doc/1001
{
  "id": 1001,
  "name": "周宁",
  "age": 20,
  "sex": "女"
}
```

指定 id 时也可以用 PUT。不指定 id、改用 `POST /reader/_doc` 时，集群生成一个 id。索引若事先不存在，第一次写入会按动态映射创建它。响应里的 `result` 为 `created`，`_version` 从 1 开始。

写入成功不等于立刻能被搜索到。文档先进内存缓冲，默认每秒刷新一次才生成可搜索的新段，所以叫近实时，不是每次 PUT 都同步可见。测试脚本如果写完马上搜，可能得到 0 条。可以在测试里调用 `POST /reader/_refresh` 强制刷新，生产索引不要对每条写入都这么做，刷新越频繁，段越多，合并压力越大。

刷新只影响搜索可见性，不保证断电后还在。落盘靠 translog 和更稀疏的 flush。每次成功的写都会追加 translog，进程崩溃后重放这段日志就能找回还没刷进段的操作。flush 把内存段写成磁盘上的提交点，并截断可以丢弃的 translog。默认由集群自己决定时机。实验里如果想观察段文件变化，可以手动 `POST /reader/_flush`，但不要放进写入接口里同步调用，否则每次请求都要等磁盘。

搜索压力大时，先看是刷新太勤导致小段过多，还是查询本身用了脚本和深度分页。`GET /reader/_stats` 里的 `refresh` 次数和 `merges` 耗时能把这两件事分开。小段多，就把刷新间隔从 1 秒调到几秒，日志场景通常可以接受。查询慢而刷新次数不高，就回到 DSL，把精确条件放进 filter，减少需要打分的命中。

同一 id 的并发写入靠版本控制。默认的内部版本在每次写成功后加 1。外部系统若自己维护版本，可以在写入时带 `version_type=external` 和版本号，版本不大于已有值的请求会被拒绝。日志采集一般不需要外部版本，重复行用不同 id 或让下游去重即可。

响应里的 `_shards` 说明这次写复制到了多少分片。`successful` 至少为 1 表示主分片已经接受。副本暂时不可用时请求仍可成功，健康会变黄，数据不会因为少了一个副本就从响应里消失。把 `successful` 理解成“集群里每一份拷贝都写完了”就会误判故障。

### 更新文档

文档在 Lucene 里是不可改的。整体覆盖用 PUT 同一 id，版本号增加，`result` 为 `updated`：

```http
PUT /reader/_doc/1001
{
  "id": 1001,
  "name": "周宁",
  "age": 21,
  "sex": "女"
}
```

局部更新走 `_update`。对外看起来只改了一个字段，内部仍是读出旧 `_source`、合并、删除旧文档、索引新文档：

```http
POST /reader/_update/1001
{
  "doc": { "age": 23 }
}
```

没带上的字段会保留。若 `_source` 被关掉，局部更新就没有旧 JSON 可以合并。

### 删除文档

```http
DELETE /reader/_doc/1001
```

`result` 为 `deleted`，版本号仍然增加。再删一次得到 404，`result` 为 `not_found`。删除不会立刻从磁盘抹掉段，只是标记，等后续合并段时才真正清掉。因此磁盘空间不会在 DELETE 返回的瞬间下降。

### 按 id 读取

```http
GET /reader/_doc/1001
```

命中时 `found` 为 true，业务字段在 `_source` 里。元数据 `_index`、`_id`、`_version` 包在外面。

### 查询全部

```http
GET /reader/_search
```

不带条件时匹配全部文档，默认只返回前 10 条。`hits.total` 在 7.x 里是一个对象，里面有 `value` 和 `relation`。不要只看这一页的长度来判断总数。

### 查询串检索

```http
GET /reader/_search?q=age:20
```

`q` 适合临时在浏览器里试。字段类型若是 text，查询串会先被分析。复杂条件用下一节的请求体，避免把逻辑堆进 URL。

### DSL 检索

查询 DSL 用 JSON 描述条件，放在 `_search` 的请求体里。

```http
POST /reader/_search
{
  "query": {
    "match": { "age": 20 }
  }
}
```

年龄大于 30 且性别匹配，可以把精确范围放进 `filter`，把需要打分的条件放进 `must`：

```http
POST /reader/_search
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "age": { "gt": 30 } } }
      ],
      "must": [
        { "match": { "sex": "女" } }
      ]
    }
  }
}
```

`filter` 不参与评分，可以被缓存。`must` 里的 match 对 text 字段会先分词。性别若是 keyword，改用 `term` 更合适。

#### 全文条件与高亮

```http
POST /reader/_search
{
  "query": {
    "match": { "name": "周宁 陈屿" }
  },
  "highlight": {
    "fields": { "name": {} }
  }
}
```

默认的 match 是“或”：两个名字中有一个词项命中即可。高亮片段出现在响应的 `highlight` 里，不改变 `_source`。

#### 聚合

聚合类似分组计数，和查询可以同时出现。对整数字段按取值分桶：

```http
POST /reader/_search
{
  "size": 0,
  "aggs": {
    "age_buckets": {
      "terms": { "field": "age" }
    }
  }
}
```

`size` 为 0 时不返回命中文档，只返回聚合。text 字段不能直接 terms 聚合，要使用 keyword 子字段。桶里的 `doc_count` 是该取值的文档数。

## 文档与元数据

一条命中记录除了业务 JSON，还有元数据。嵌套对象直接写在 `_source` 里：

```json
{
  "_index": "reader",
  "_id": "1005",
  "_version": 1,
  "_score": 1.0,
  "_source": {
    "id": 1005,
    "name": "宋晚",
    "age": 37,
    "card": { "cardNo": "C-8831" }
  }
}
```

### 三个必须能对上的概念

- 索引名指出文档属于哪一组。程序只和索引打交道，文档实际落在哪个分片由集群计算，调用方不用选择分片。
- 类型在 7.x 固定为 `_doc`。旧版本里类型用来区分同一索引中的不同结构，并各自带映射；同名字段仍必须同类型。类型名不能包含下划线开头的保留形式以外的某些字符，新索引不要再设计类型名。
- `_id` 是索引内的字符串主键。可以自己指定，也可以让集群生成。它和索引名一起唯一确定一篇文档。

### 让响应更好读

URL 加上 `?pretty=true`，JSON 会缩进。这只影响展示，不改变内容。

只要部分字段时使用 `_source` 过滤：

```http
GET /reader/_doc/1005?_source=id,name
```

只要原始 JSON、不要外层元数据：

```http
GET /reader/_source/1005
```

再加过滤：

```http
GET /reader/_source/1005?_source_includes=id,name
```

### 判断是否存在

```http
HEAD /reader/_doc/1005
```

存在返回 200，不存在返回 404，没有正文。这个结果只代表请求到达的那一刻。另一个客户端可能紧接着写入同一 id。

## 批量操作

多次往返可以收成一次批量请求，减少网络开销。

### 多 id 读取

```http
POST /reader/_mget
{
  "ids": ["1001", "1003"]
}
```

某一条不存在时，整体仍是 200，该条的 `found` 为 false。调用方要逐条看 `found`，不能因为 HTTP 成功就认为全部命中。

### _bulk

批量写入的正文不是一个 JSON 数组，而是换行分隔的操作行和可选的来源行。最后一行要有换行。

```http
POST /_bulk
{"index":{"_index":"reader","_id":"2001"}}
{"id":2001,"name":"江禾","age":20,"sex":"男"}
{"index":{"_index":"reader","_id":"2002"}}
{"id":2002,"name":"江禾二","age":21,"sex":"女"}
{"delete":{"_index":"reader","_id":"2001"}}
```

`index` 和 `create` 下一行是文档；`delete` 和只更新元数据的操作没有来源行，下一行直接是下一条操作。响应里 `errors` 为 true 时，要看每条的 `error`，允许部分成功。

一次 bulk 会完整加载到接收节点的内存里。太大就会挤占其他请求。起点可以放在每批 1000 到 5000 篇，同时看字节数：一批控制在数 MB 到十几 MB 更稳。文档很大时减少篇数。性能开始变差，说明已经超过这台机器的合适批次。

## 分页

`from` 是偏移，`size` 是本页条数，默认 0 和 10。

```http
GET /reader/_search?size=5
GET /reader/_search?size=5&from=5
GET /reader/_search?size=5&from=10
```

### 深分页为什么贵

假设索引有 5 个主分片，要第 1 到第 10 条。每个分片先产生自己的前 10 条，协调节点排序这 50 条，留下 10 条。

要第 10001 到第 10010 条时，每个分片必须先产生自己的前 10010 条。协调节点排序大约 50050 条再丢掉绝大部分。成本随页码线性上涨，所以搜索引擎通常不允许无限制地翻到很深。真正要遍历全量时用 `search_after` 或滚动查询，而不是把 `from` 加到很大。

## 映射

动态映射按第一批值猜测。字符串会变成 text，并带一个 keyword 子字段；整数变成 long 或 integer；布尔和日期也有对应规则。猜测适合试数据，不适合要聚合的生产索引。

7.x 用 text 和 keyword 取代旧的 string：

- text：要全文检索的字段，如简介、标题。写入前按分析器切成词项。不适合排序和聚合。
- keyword：不分词，适合状态、邮箱、标签、精确过滤、排序和聚合。

动态映射常见的猜测可以记成一张对照，正式索引仍建议显式写出，避免第一批脏数据把类型定死：

| 写入的 JSON | 猜成的类型 | 注意 |
| --- | --- | --- |
| 带引号的文本 | text，并附加 keyword 子字段 | 聚合要用 `.keyword` |
| 整数字面量 | long | 年龄一类用 integer 更省 |
| 小数字面量 | float 或 double | 金额用 `scaled_float` 并指定 `scaling_factor` |
| true / false | boolean | 不要用 `"true"` 字符串 |
| 符合日期格式的字符串 | date | 格式要和后续写入一致 |
| 对象 | object | 内部字段被展平进同一篇文档的映射 |
| 对象数组 | object | 数组元素的字段会交叉匹配，需要改用 nested |

除此之外还可以显式声明：`byte`、`short`、`integer`、`half_float`、`date_nanos`、`binary`、`ip`、`geo_point`，以及整数范围、日期范围这类 range。`binary` 不参与检索，只在 `_source` 里保存。`ip` 支持 CIDR 查询。地理点用经纬度对象或字符串，才能做距离排序。

object 和 nested 的差别在实验里很容易被忽略。两名联系人写成对象数组时，object 映射会把所有 `name` 和所有 `phone` 摊平，查询可能把张三的姓名和李四的电话配成一对。nested 把每个数组元素当成隐藏的独立文档，查询要用 `nested` 子句指明路径。nested 更准，写入和查询也更贵，只在“数组元素内部的字段必须一起命中”时使用。

映射一旦发布就不能把 text 改成 keyword，也不能把 long 改成 text。能做的是新增字段，或给已有字段加多字段（fields）。改错类型的唯一稳妥办法是新建索引、用 `_reindex` 把旧数据拷过去，再把别名切到新索引。别名这一步让查询方不用改索引名。练习时直接删索引重建更快，但那会丢掉文档。

### 显式创建

```http
PUT /shelf
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name":  { "type": "text" },
      "age":   { "type": "integer" },
      "mail":  { "type": "keyword" },
      "hobby": { "type": "text" }
    }
  }
}
```

查看映射：`GET /shelf/_mapping`。随后用 bulk 写入几条读者，爱好用顿号连成一段中文，便于后面的分词实验。邮箱用 keyword，查询时必须整段匹配，不要对它做 match。

### 试一次全文条件

```http
POST /shelf/_search
{
  "query": {
    "match": { "hobby": "音乐" }
  }
}
```

标准分析器对中文几乎按字切开，召回会偏杂。等装上中文分词器后再为 `hobby` 指定分析器，同样的查询才按词而不是按字命中。

## 结构化查询

### term

`term` 按倒排索引里的精确词项匹配，适合数字、日期、布尔和 keyword。它不会对查询词再做分词。

```http
POST /shelf/_search
{
  "query": { "term": { "age": 20 } }
}
```

对 text 字段使用 term，常常什么都查不到，因为索引里存的是分析之后的词项，不是原始句子。

### terms

`terms` 给出多个精确值，命中其中任意一个即可，相当于同一字段上的或。

```http
POST /shelf/_search
{
  "query": { "terms": { "age": [20, 21] } }
}
```

### range

范围运算符有 `gt`、`gte`、`lt`、`lte`。

```http
POST /shelf/_search
{
  "query": {
    "range": { "age": { "gte": 20, "lte": 22 } }
  }
}
```

日期字段同样可用，字符串要符合映射里的格式。

### exists

查找某字段有值的文档，接近 SQL 里的 `IS NOT NULL`。字段缺失、值为 null，或没有被索引时，不会命中。

```http
POST /reader/_search
{
  "query": { "exists": { "field": "card" } }
}
```

### match

`match` 是常用的查询入口。目标是 text 时，查询串会先经过同一套分析器，再按词项检索。目标是数字、日期、布尔或 keyword 时，它按给定值去匹配，表现接近 term。

```json
{ "match": { "hobby": "游泳 音乐" } }
{ "match": { "age": 26 } }
```

### bool

`bool` 把多条子句组合起来：

- `must`：都要满足，参与评分，类似且
- `must_not`：都不能满足，不参与评分
- `should`：满足得越多分越高；若没有 `must`，默认至少要满足一条
- `filter`：都要满足，不评分

```json
{
  "bool": {
    "must":     { "term": { "folder": "inbox" } },
    "must_not": { "term": { "tag": "spam" } },
    "should": [
      { "term": { "starred": true } },
      { "term": { "unread": true } }
    ]
  }
}
```

## 过滤与查询的差别

把 `term` 放进 `bool.filter`，得到的是“年龄是否等于 20”这种是非判断，不计算相关度：

```http
POST /shelf/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "age": 20 } }
      ]
    }
  }
}
```

查询子句要问相关程度，并为每篇文档计算 `_score`，再按分数排序。过滤子句只产生匹配集合，位图可以被缓存，后续请求能复用。既要精确条件又要全文排序时，精确条件放 `filter`，全文放 `must`。纯精确匹配不必用 match 去算分。

## 中文分词

### 分词在做什么

分析是把一段文本变成一串词项，供倒排索引使用。英文可以靠空格切开，“我是中国人”没有这种天然边界，切成“中国”还是“国人”会直接影响能不能被查到。

### 分析接口

```http
POST /_analyze
{
  "analyzer": "standard",
  "text": "hello world"
}
```

响应里每个 token 带有词面和在原文中的起止偏移。对某个索引的字段做分析：

```http
POST /shelf/_analyze
{
  "field": "hobby",
  "text": "听音乐"
}
```

这样用的是该字段映射里的分析器，而不是临时指定的另一套，结果才和查询时一致。

### 中文的难点

同一句话可以有多种切法，切错就会产生歧义。常见的中文分词组件有 IK、jieba 等。和 Elasticsearch 配套、使用最广的是 IK 的插件。它结合词典和规则，能处理中英混排、数字和常见专名。词典决定了未登录词会不会被切开，业务词要能补进词典。

### 安装 IK

插件版本必须和 Elasticsearch 版本完全一致。把发行包解压到 `plugins/ik`，目录属主改为运行用户，然后重启进程。

```bash
su - observe
cd /opt/observe/elasticsearch
./bin/elasticsearch-plugin install \
  https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.9/elasticsearch-analysis-ik-7.17.9.zip
```

离线时改为把 zip 解压到 `plugins/ik`。启动日志里出现 ik 的插件名，才算加载成功。不要把 7.9 的插件装进 7.17 的节点。

### 两种粒度

```http
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "我是中国人"
}
```

`ik_max_word` 会尽量切出更多词，适合索引阶段，提高召回。`ik_smart` 切得粗，适合搜索阶段，减少不必要的词项。索引和查询可以使用不同分析器，但要成对设计，否则索引里的词和查询里的词对不上。

## 全文检索

相关度用来给命中排序。默认的实用打分会考虑词频（词在这篇文档里出现多少次）、逆文档频率（词在整个索引里有多罕见）以及字段长度（字段越短，同样的命中越显眼）。分析则负责在建索引和查询时把文本变成同一套词项。两者缺一不可：没有分析，中文匹配不准；没有相关度，结果只能按写入顺序堆在一起。

### 准备带中文分析器的索引

已经存在的 `shelf` 若分析器不对，删掉重建。text 字段的分析器不能在有数据后随意修改。

```http
PUT /shelf
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name":  { "type": "text" },
      "age":   { "type": "integer" },
      "mail":  { "type": "keyword" },
      "hobby": { "type": "text", "analyzer": "ik_max_word" }
    }
  }
}
```

写入五条文档。爱好覆盖羽毛球、篮球、游泳、音乐、跑步的不同组合，姓名用周宁、陈屿、林深、赵衡、宋晚。邮件保持 keyword，方便后面用 term 精确比对。

```http
POST /_bulk
{"index":{"_index":"shelf","_id":"s1"}}
{"name":"周宁","age":20,"mail":"ning@example.com","hobby":"羽毛球、乒乓球"}
{"index":{"_index":"shelf","_id":"s2"}}
{"name":"陈屿","age":21,"mail":"yu@example.com","hobby":"羽毛球、篮球"}
{"index":{"_index":"shelf","_id":"s3"}}
{"name":"林深","age":22,"mail":"shen@example.com","hobby":"篮球、游泳、音乐"}
{"index":{"_index":"shelf","_id":"s4"}}
{"name":"赵衡","age":23,"mail":"heng@example.com","hobby":"跑步、游泳"}
{"index":{"_index":"shelf","_id":"s5"}}
{"name":"宋晚","age":24,"mail":"wan@example.com","hobby":"音乐、羽毛球"}
```

bulk 的响应里 `errors` 应为 false。若 IK 没装好，索引创建这一步就会失败，而不是等到查询。失败信息在对应条目的 `error.reason` 里，常见的是 analyzer `ik_max_word` 不存在。装好插件并重启节点后，删除半成品索引再执行上面的 PUT 和 bulk。文档数用 `GET /shelf/_count` 确认是 5，再开始下面的查询，避免把“没写进去”误判成分词结果不对。

### 单词查询的过程

查询 `hobby` 为“音乐”时，内部大致四步：

1. 看映射：`hobby` 是 text，并且指定了 IK，查询串也要走这套分析。
2. 分析“音乐”，得到词项。
3. 用该词项查倒排，得到包含它的文档 id。
4. 按词频、逆文档频率和字段长度计算 `_score`。

加上 `highlight` 后，命中的爱好字段会带回带标记的片段。标记默认是 `em` 标签，只用于展示。

### 多词查询

`match` 默认按或组合词项，查“音乐 篮球”会召回只含其中一个的文档。若两个都要，指定 `operator` 为 `and`：

```json
{
  "query": {
    "match": {
      "hobby": {
        "query": "音乐 篮球",
        "operator": "and"
      }
    }
  }
}
```

或与且是两端。中间状态用 `minimum_should_match`，可以是个数或百分比。例如“游泳 羽毛球”在 80% 和 40% 下召回的文档数不同。百分比按分析之后的词项数计算，中文分词粒度会改变分母，所以这个值要拿真实语料试，不能从别的语言的经验直接搬。

### 组合条件

bool 可以把“必须、必须没有、有则加分”写在一起：

```json
{
  "query": {
    "bool": {
      "must":     { "match": { "hobby": "篮球" } },
      "must_not": { "match": { "hobby": "音乐" } },
      "should":   [ { "match": { "hobby": "游泳" } } ]
    }
  }
}
```

含义是：爱好里要有篮球，不能有音乐；若还有游泳，分数更高。评分把命中的 `must` 和 `should` 分数相加，再除以这些子句的数量。`must_not` 只排除，不加分。

只有 `should`、没有 `must` 时，默认至少命中一条。可以用 `minimum_should_match` 提高门槛，例如三条爱好里至少命中两条：

```json
{
  "bool": {
    "should": [
      { "match": { "hobby": "游泳" } },
      { "match": { "hobby": "篮球" } },
      { "match": { "hobby": "音乐" } }
    ],
    "minimum_should_match": 2
  }
}
```

### 权重

`boost` 提高某个子句的权重。搜索必须同时包含“游泳”和“篮球”，同时若包含“音乐”则明显提前，包含“跑步”只稍微提前：

```json
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "hobby": { "query": "游泳 篮球", "operator": "and" }
        }
      },
      "should": [
        { "match": { "hobby": { "query": "音乐", "boost": 10 } } },
        { "match": { "hobby": { "query": "跑步", "boost": 2 } } }
      ]
    }
  }
}
```

权重是相对的。所有子句都放大同样的倍数，排序不会变。它也不能代替正确的分析器。

调试打分时可以在查询上临时加 `"explain": true`。响应会给出每个子句的词频、逆文档频率和字段长度归一如何合成最终 `_score`。解释树很长，只在一两篇文档上打开，不要对全量查询默认开启。确认排序符合预期后关掉，避免协调节点多做一次计算。

同一查询在分片数不同的两个索引上，分数不可直接比较。逆文档频率是在分片本地统计的，分片一切，罕见词的统计基数就变了。需要跨索引比分数时，再考虑 `dfs_query_then_fetch` 先汇总词频。日常日志检索用过滤加上时间排序即可，不必追求分数在分片之间严格一致。

## 集群

### 节点角色

同一 `cluster.name` 的节点才会组成一个集群，`node.name` 在集群内用来区分彼此。

- 主节点资格：`node.master` 为 true 的节点可以参加选举。主节点负责创建或删除索引、跟踪分片落在哪一台机器上，不必须亲自存数据。
- 数据节点：`node.data` 为 true 时保存分片，执行索引和查询。
- 协调节点：两个属性都为 false 时，不参选、不存数据，只接收请求并转发给其他节点。
- 部落节点是更早版本里用 `tribe.*` 连接多个集群的客户端角色，7.x 已移除，跨集群搜索改用专门的跨集群配置，不要再写部落节点。

默认情况下一个节点既有主节点资格也存数据。实验可以这样，规模变大后应把主节点和数据节点分开，避免重查询影响选举。

### 三台节点

在 7.x 上不要再写 `discovery.zen.ping.unicast.hosts` 和 `discovery.zen.minimum_master_nodes`，这两项属于 6.x。7.x 用种子主机列表，第一次建群时用 `cluster.initial_master_nodes`。

三台机器的 `cluster.name` 都设为 `campus-cluster`，`node.name` 分别是 `n1`、`n2`、`n3`，`network.host` 设为各自的内网地址。种子列表三台都写全：

```yaml
cluster.name: campus-cluster
node.name: n1
node.master: true
node.data: true
network.host: 10.0.8.11
http.port: 9200
discovery.seed_hosts: ["10.0.8.11", "10.0.8.12", "10.0.8.13"]
cluster.initial_master_nodes: ["n1", "n2", "n3"]
```

另外两台只改 `node.name` 和 `network.host`。`initial_master_nodes` 里的名字必须是节点名，不是 IP。集群一旦选过主节点，就不要在每次重启时改这份列表。

head 或浏览器需要跨域时，再按前面的 CORS 配置写上具体来源。三台都启动后，`GET /_cluster/health` 查看状态。

健康颜色：

- green：所有主分片和副本都已分配
- yellow：主分片都在，至少有一个副本没分配。单机把副本数设成 1 时就会一直黄
- red：有主分片没分配，部分数据不可读

`GET /_cluster/health` 的 JSON 比颜色多几项，排错时要一起看：

- `cluster_name`：必须是三台配置里的同一个名字，否则根本不是这个集群
- `number_of_nodes` 与 `number_of_data_nodes`：节点没到齐时，先查种子列表和防火墙是否放通 9300，HTTP 的 9200 通了不代表传输口通
- `active_primary_shards`：比 `active_shards` 小，差值是副本。主分片数掉下去就是红
- `unassigned_shards`：大于 0 且没有正在初始化的分片时，看 `GET /_cluster/allocation/explain`，常见原因是磁盘水位、副本无处可放，或索引创建时分片数大于节点所能承载的副本布局
- `status` 为 yellow 而 `unassigned_shards` 等于副本总数，在单节点上是正常现象，把副本调成 0 就会变绿

不要用反复重启来“等它变绿”。未分配的原因写在 explain 接口里。磁盘超过高水位时，节点会拒绝新副本，日志里是 flood-stage 或 watermark，清出空间或调整水位后再等分配，而不是删掉 data 目录。

索引建好后可以用 `GET /_cat/shards/shelf?v` 看每一片落在哪个节点。主分片标记为 `p`，副本为 `r`。三台都在、副本为 1 时，每一片的 p 和 r 不应出现在同一节点。若出现，说明感知到的节点数其实是 1，集群名或种子主机仍有一台没加入。

### 分片与副本

索引只是逻辑名字。分片才是 Lucene 的一个索引实例，也是搬迁和复制的最小单位。应用不直接访问分片。

每篇文档只属于一个主分片。主分片数量决定了索引能水平拆到多细，创建后不能更改，因为路由公式用到了这个数。副本是主分片的拷贝，用来在节点失败时顶上，也可以承担读请求。副本数可以随时调整：

```http
PUT /shelf/_settings
{
  "number_of_replicas": 1
}
```

三节点集群里，1 个副本通常能分配成功，健康变为绿色。

### 故障转移

停掉一台数据节点后，健康先变黄：主分片还在，原本放在这台机器上的副本丢失。集群会把缺失的副本在剩余节点上补出来，完成后回到绿色。把节点重新启动，它以原有身份加入，分片会再次均衡。

停掉当前主节点时，其余具备资格的节点重新选举。选举期间可能短暂黄或红，新的主节点确定并且分片补齐后恢复绿色。7.x 使用投票配置避免两个节点同时认为自己是主节点。6.x 靠 `minimum_master_nodes` 设为法定人数（大约 N/2+1）来避免脑裂；设得太小，旧主节点恢复后可能无法加入，集群被分成两半。7.x 不要再配这个键，改坏投票配置才会重新遇到类似现象。

### 文档写到哪一片

路由公式：

```text
shard = hash(routing) % number_of_primary_shards
```

`routing` 默认是 `_id`，也可以在写入时指定。哈希后对主分片数取余，余数就是分片序号。主分片数若在中途改变，同一 id 会算出另一片，旧数据就找不到了，所以主分片数必须在创建时定死。

### 写入路径

新建、覆盖和删除都是写操作，必须先在主分片成功，再复制到副本。

1. 客户端把请求发给任意节点，该节点成为协调节点。
2. 协调节点按 id 算出主分片，把请求转给持有该主分片的节点。
3. 主分片执行成功后，并行转发到副本。副本都确认后，主分片节点向协调节点报告成功，再返回客户端。

客户端收到成功时，主分片和当时的副本都已经应用了这次修改。

### 按 id 读取

读可以打到主分片或任一副本。

1. 协调节点算出分片序号。
2. 在持有该分片主副本的节点里选一台，把 GET 转过去，用来摊开读压力。
3. 那台节点把文档返回协调节点，再返回客户端。

副本可能略滞后于主分片。索引请求一旦向客户端返回成功，之后的读就能在主分片和副本上拿到文档。在此之前读到副本，有可能暂时 404。

### 分布式全文检索

检索分查询和取回两个阶段。

查询阶段：

1. 协调节点收到搜索请求，准备一个长度为 `from + size` 的优先队列。
2. 请求被转发到每个分片的主分片或某个副本。分片在本地查询，保留自己的前 `from + size` 条。
3. 分片只回文档 id 和排序值。协调节点合并成全局的前 `from + size` 条。

取回阶段：

1. 协调节点知道最终要哪些 id，向保存它们的分片发 GET。
2. 分片加载 `_source` 或指定字段，返回协调节点。
3. 协调节点按全局顺序组装响应，交给客户端。

所以深分页在查询阶段就要让每个分片准备很长的队列，问题出在合并之前，不是出在最后那一页的网络传输。

别名让读写分离到不同的物理索引上。创建 `shelf_v1` 后执行：

```http
POST /_aliases
{
  "actions": [
    { "add": { "index": "shelf_v1", "alias": "shelf_read" } }
  ]
}
```

查询打到 `shelf_read`，重建时把数据 `_reindex` 到 `shelf_v2`，再用一次别名切换把读指向新索引，中间可以短暂地把别名同时指向两边做校验。应用配置里只写别名，不写带版本的物理索引名。删除物理索引之前先确认别名已经挪走，否则查询会直接 404。

分析器由三截组成，映射里写的 `ik_max_word` 是一整条链的名字。字符过滤先做大小写或全半角折叠，分词器再切开，词元过滤最后去掉停用词或做同义词。自定义分析器时这三截都要在 settings 里声明，再在字段上引用。只改查询时的分析器、不改索引时的分析器，倒排里的词项和查询词对不上，表现就是“样例文本能被 `_analyze` 切开，搜索却没有命中”。排查时对同一段文本分别指定索引分析器和搜索分析器各跑一次 `_analyze`，比较 token 列表是否一致。

数字字段做范围查询比把数字存成 keyword 再脚本转换便宜得多。能在写入前定成 integer、long 或 scaled_float 的，不要等聚合脚本去补救。日期统一用一种格式写入，映射里的 `format` 写清楚，避免同一字段混入时间戳毫秒和 `yyyy-MM-dd` 字符串，动态映射会在第二种格式出现时直接拒绝文档。

## Java 客户端

7.x 官方推荐 REST 客户端。传输层的 Java API 客户端随节点版本绑定得更紧，新代码不要再从它入手。REST 客户端分低级和高级两层。

- 低级客户端：只负责 HTTP。调用方自己拼 JSON、自己解析响应。对集群版本的要求宽松。
- 高级客户端：在低级客户端之上提供索引、查询、更新的请求对象，少写 JSON。版本要和集群主版本匹配。

下面的依赖使用 7.17 的客户端对准 7.17 的集群。6.8 的客户端连 7.x 时，类型参数和请求路径都会对不上。

### 样例数据

索引 `lodging`，文档是一间房间的编号、标题和价格。用 bulk 写入六条，标题里混一些可被中文分词命中的词，例如“近地铁”“可入住”。价格用整数分而不是带货币符号的字符串，避免映射被猜成 text。

```http
POST /_bulk
{"index":{"_index":"lodging","_id":"r1"}}
{"title":"近地铁 单间 可入住","price":7500}
{"index":{"_index":"lodging","_id":"r2"}}
{"title":"湖景 一室 可入住","price":8500}
```

其余四条按同样格式补上不同标题和价格即可。

### 低级客户端

依赖：

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-client</artifactId>
    <version>7.17.9</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.5</version>
</dependency>
```

客户端只保存主机和端口，请求对象自己指定方法和路径：

```java
public class LowLevelLab {
    private RestClient client;
    private final ObjectMapper mapper = new ObjectMapper();

    public void open() {
        client = RestClient.builder(
                new HttpHost("127.0.0.1", 9200, "http")).build();
    }

    public void clusterState() throws IOException {
        Request request = new Request("GET", "/_cluster/health");
        request.addParameter("pretty", "true");
        Response response = client.performRequest(request);
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    public void createRoom() throws IOException {
        Request request = new Request("POST", "/lodging/_doc");
        Map<String, Object> room = new HashMap<>();
        room.put("title", "西区 两室 近地铁");
        room.put("price", 6400);
        request.setJsonEntity(mapper.writeValueAsString(room));
        Response response = client.performRequest(request);
        System.out.println(response.getStatusLine());
    }

    public void search() throws IOException {
        Request request = new Request("POST", "/lodging/_search");
        request.setJsonEntity("{\"query\":{\"match\":{\"title\":\"近地铁\"}}}");
        System.out.println(EntityUtils.toString(
                client.performRequest(request).getEntity()));
    }
}
```

低级客户端的好处是路径完全自己控制，升级集群时不容易被高层 API 的方法签名卡住。代价是查询 DSL 以字符串形式出现，编译器帮不上忙。用完要 `close()`，否则连接池留在进程里。

### 高级客户端

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.17.9</version>
</dependency>
```

高级客户端内部持有低级客户端。7.x 的索引请求不再强制类型参数，构造 `IndexRequest` 时只给索引名。

同步写入：

```java
public void indexRoom(RestHighLevelClient client) throws IOException {
    Map<String, Object> room = new HashMap<>();
    room.put("title", "东区 一室 可入住");
    room.put("price", 4500);
    IndexRequest request = new IndexRequest("lodging").id("r9").source(room);
    IndexResponse response = client.index(request, RequestOptions.DEFAULT);
    System.out.println(response.getId() + " " + response.getResult());
}
```

异步写入把同一个请求交给 `indexAsync`，在 `ActionListener` 的 `onResponse` 里读结果，在 `onFailure` 里记失败原因。调用异步方法后当前线程会立刻返回，示例里若马上结束进程，回调还没执行连接就关了，需要等回调或使用可关闭的门闩。

按 id 读取并限制字段：

```java
GetRequest get = new GetRequest("lodging", "r9");
get.fetchSourceContext(new FetchSourceContext(
        true, new String[]{"title", "price"}, Strings.EMPTY_ARRAY));
GetResponse response = client.get(get, RequestOptions.DEFAULT);
```

存在性检查把 `FetchSourceContext` 设为不返回 `_source`，只看 `client.exists(...)`。删除使用 `DeleteRequest`，状态是 OK 或 NOT_FOUND。局部更新使用 `UpdateRequest`，`doc` 里只放要改的字段，响应里的版本号会增加。

检索：

```java
SearchRequest search = new SearchRequest("lodging");
SearchSourceBuilder source = new SearchSourceBuilder()
        .query(QueryBuilders.matchQuery("title", "可入住"))
        .from(0)
        .size(5)
        .timeout(new TimeValue(5, TimeUnit.SECONDS));
search.source(source);
SearchResponse response = client.search(search, RequestOptions.DEFAULT);
for (SearchHit hit : response.getHits()) {
    System.out.println(hit.getSourceAsString());
}
```

`from` 和 `size` 的深分页成本和 REST 相同，客户端不会在本地把问题消掉。超时只限制这次查询在协调节点上的等待，不是事务超时。

局部更新只提交变化的字段，避免把整篇文档在客户端再写一遍：

```java
public void changePrice(RestHighLevelClient client) throws IOException {
    Map<String, Object> patch = new HashMap<>();
    patch.put("price", 6900);
    UpdateRequest request = new UpdateRequest("lodging", "r9").doc(patch);
    UpdateResponse response = client.update(request, RequestOptions.DEFAULT);
    System.out.println(response.getVersion());
}
```

删除与存在性检查：

```java
public void removeIfPresent(RestHighLevelClient client) throws IOException {
    GetRequest probe = new GetRequest("lodging", "r9");
    probe.fetchSourceContext(new FetchSourceContext(false));
    if (!client.exists(probe, RequestOptions.DEFAULT)) {
        return;
    }
    DeleteResponse deleted = client.delete(
            new DeleteRequest("lodging", "r9"), RequestOptions.DEFAULT);
    System.out.println(deleted.status());
}
```

异步写入适合批量导入时不想堵住调用线程的场景。监听器里不要再做耗时的阻塞调用，失败时记录 id，方便之后重试，而不是把异常吞掉：

```java
public void indexLater(RestHighLevelClient client, Map<String, Object> room) {
    IndexRequest request = new IndexRequest("lodging").source(room);
    client.indexAsync(request, RequestOptions.DEFAULT, new ActionListener<IndexResponse>() {
        @Override
        public void onResponse(IndexResponse response) {
            System.out.println(response.getResult());
        }
        @Override
        public void onFailure(Exception ex) {
            System.out.println(ex.getMessage());
        }
    });
}
```

`RestHighLevelClient` 内部的连接池是线程安全的，整个进程保持一个实例即可，不要按请求新建。它在 7.15 之后进入维护，新的官方 Java API Client 用更接近 JSON 的请求对象。已经写在 7.17 上的高级客户端可以继续用到配套的 7.17 集群；升级到 8.x 时再换客户端，不要只升一边。

集群如果开启了内置安全，低级和高级客户端都要补上凭据和证书。实验集群没开安全时不要在示例里写死口令。把主机名放进配置而不是写进代码，同一套客户端才能从单机改到三节点：`RestClient.builder` 接受多个 `HttpHost`，协调工作由集群完成，客户端轮流尝试这些地址即可。
