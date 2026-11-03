# Kibana 入门

Kibana 是 Elastic Stack 里的查询和可视化组件，用来搜索 Elasticsearch 里的索引、做图表，并把多张图放进仪表盘。它自己不存业务数据，浏览器上的每一次探索最终都变成发往 Elasticsearch 的检索请求。

## 配置和安装

选择与 Elasticsearch 主版本一致的发行包。在实验机上解压并改成短目录名：

```bash
tar -xzf kibana-7.17.9-linux-x86_64.tar.gz
mv kibana-7.17.9-linux-x86_64 /opt/observe/kibana
```

编辑 `config/kibana.yml`，至少改两处。`server.host` 决定 Kibana 监听哪块网卡；只写回环地址时，别的机器打不开页面。`elasticsearch.hosts` 指向集群的 HTTP 地址。7.x 里这项的键名是 `elasticsearch.hosts`，更早的文档里写作 `elasticsearch.url`，照旧键名写会在启动日志里看到未识别配置。

```yaml
server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["http://127.0.0.1:9200"]
```

## 启动

Kibana 不允许用 root 直接跑。给运行用户准备家目录和目录所有权后再启动：

```bash
chown -R observe:observe /opt/observe
su - observe
/opt/observe/kibana/bin/kibana
```

进程就绪后访问 `http://<主机>:5601`。若页面一直转圈，先确认 Elasticsearch 的 9200 能通，再看 Kibana 日志里是证书、口令还是地址写错。实验环境若没开安全特性，不要在配置里提前填写生产用的账号。

## 功能说明

7.x 左侧常见入口可以按用途分组记忆：

- Discover：按时间和字段翻原始文档
- Visualize：做柱状图、饼图、折线等单张图
- Dashboard：把多张图拼成一屏
- Canvas：做更自由的展示板
- Machine Learning：异常检测一类的分析作业
- Logs / Metrics / APM：分别看日志、主机指标和应用性能，数据要先由对应的 Beat 送进来
- Dev Tools：直接写 REST 请求
- Stack Monitoring：看 Elasticsearch 与 Kibana 自身的健康
- Management：索引模式、生命周期、安全等管理项

Timelion 在较早版本里是独立的时序表达式入口，后来能力并进了 Lens 和时序可视化，不必再单独找旧菜单。

## 数据探索

Discover 不会自动知道该搜哪个索引。先在 Management 里创建 index pattern，用通配符把一组按天滚动的索引收进来，例如 `gate-access-*`。向导会请你指定一个时间字段，日志场景选文档里的 `@timestamp` 或业务时间，这样右上角的时间选择器才能切片。

模式建好后，往采集源追加一行，例如：

```bash
echo "gate refused token=demo-7" >> /var/log/gate/error.log
```

回到 Discover 刷新。新文档应出现在时间直方图下方。左侧字段列表可以勾选要显示的列，避免整段 JSON 铺满屏幕。时间选择器默认常常是最近 15 分钟，日志的时间戳若是旧的，会表现为“索引有数据但页面是空的”，先把范围拉大再判断采集是否失败。

## Metricbeat 仪表盘

指标要在 Kibana 里看到现成仪表盘，需要让 Metricbeat 知道 Kibana 的地址，并把仪表盘资源导入进去。采集进程和导入过程是两件事。

```yaml
setup.kibana:
  host: "http://10.0.4.20:5601"
```

```bash
./metricbeat setup --dashboards
./metricbeat -e
```

`setup` 会写入索引模板和一组 Dashboard 对象，耗时取决于机器。完成后在 Dashboard 列表里按 Metricbeat 过滤，就能打开系统 CPU、内存、进程等预制图。图是空的通常说明时间范围内没有新样本，或 Beat 还没连上 Elasticsearch。

## 网关指标仪表盘

Metricbeat 的 nginx 模块开启并成功采集后，Dashboard 里会多出对应的指标板，展示活动连接、接受连接数和处理的请求数。这张板读的是指标文档，不是访问日志原文。模块没启用、或者网关没有打开状态页时，板会在但序列是空的。

## 网关日志仪表盘

访问日志走 Filebeat。可以复用模块自带的解析和仪表盘，而不是手写 Grok。准备一份只负责模块和输出的配置，例如 `gate-access.yml`：

```yaml
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
  host: "http://127.0.0.1:5601"
output.elasticsearch:
  hosts: ["http://127.0.0.1:9200"]
```

先导入对象，再启动采集：

```bash
./filebeat -c gate-access.yml setup
./filebeat -e -c gate-access.yml
```

回到 Dashboard，打开 Filebeat 的 nginx 相关面板，就能看到状态码分布、来源和路径统计。这些面板不是 Kibana 安装包自带的，是 `setup` 阶段由 Filebeat 写进去的。换一套索引名却不重新 setup，面板仍指向旧索引，图也会是空的。

## Kibana 自定义仪表盘

预制板不够用时，在 Visualize 里新建。选 Vertical Bar，索引模式选刚才的日志模式。纵轴用 Count，横轴用 Date Histogram，间隔跟你的时间选择器匹配。保存之后，到 Dashboard 里 Add 这张图。

饼图适合类别占比。Buckets 里选 Split Slices，字段用 keyword 类型的动作名或状态码。text 字段做了分词，不能直接当饼图维度，需要使用它的 `.keyword` 子字段。给每个切片写上自定义标签后，图例才可读。

数据表可视化则勾选要展示的字段，适合和柱状图放在同一块仪表盘里互相下钻。仪表盘上的时间选择器和筛选条会同时作用到其中每一张图。

## 开发者工具

Dev Tools 的 Console 相当于一个带补全的 REST 客户端，请求直接打到当前连接的 Elasticsearch。查映射、试查询、看分词，都不必再另开一个 HTTP 工具。例如确认索引是否存在：

```http
GET gate-access-*/_mapping
```

左侧写请求，右侧看 JSON。这里执行的删除和改映射会真实作用到集群，实验索引和生产索引不要放在同一个 Console 历史里混着点。
