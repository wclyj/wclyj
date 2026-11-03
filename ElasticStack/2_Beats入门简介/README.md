# Beats 入门

用 Filebeat 和 Metricbeat 收集反向代理的访问日志和运行指标，写入 Elasticsearch，供 Kibana 查看。

## 需求

反向代理常常是站点的唯一入口。它一旦拒绝连接或错误率升高，后面的应用看起来也像全部不可用。需要两份材料：持续追加的访问日志，以及定时采样的连接数、请求数。两者都要进同一套检索和图表，而不是靠登录机器尾随文件。

## 数据怎么走

采集器安装在跑代理的那台机器上。Filebeat 读日志文件，Metricbeat 拉状态页。事件直接进入 Elasticsearch。Kibana 只读索引并出图。

若日志行还要切割字段，就不要直写 Elasticsearch，先送到 Logstash，加工后再入库。指标文档结构已经由模块定好，通常不需要再绕 Logstash。

## 代理本身

代理按发行版装好并启动后，访问日志一般在 `/var/log/nginx/access.log`。先在本机确认文件会增长：

```bash
tail -f /var/log/nginx/access.log
```

文件不增长时，后面的采集器不会有输入。错误日志路径一并记下，Filebeat 的模块会同时收集它。

## Beats 是什么

Beats 是一组单一用途的采集程序，用同一套输出配置把事件送到 Logstash 或 Elasticsearch。和在每台机器上跑一个 JVM 管道相比，它们的内存占用小，适合铺到大量主机。

和本篇相关的三个：

- Filebeat：转发日志文件
- Metricbeat：按周期拉指标
- Packetbeat：从网卡抓应用层流量，解析 HTTP、数据库等协议

不需要加工的事件直接写 Elasticsearch。需要解析、脱敏或路由的事件先写 Logstash。Kibana 不接收 Beats 的输出，它只查询已经入库的文档。

## Filebeat

### 它解决什么

面对几十台上百台主机的日志，逐台 SSH 去看文件不可持续。Filebeat 盯住本地文件，把新增的行送出去，并记住送到了哪一行。它是日志转发器，不是检索引擎。

记住两点：进程要轻；出口是 Elasticsearch 或 Logstash，图在 Kibana 里看。

### 内部路径

一条配置可以声明多个输入。输入用通配符匹配文件。每个被选中的文件由一个 harvester 逐行读取，读到的事件交给发布管道，再按输出配置送到目的地。旧文档里的 spooler 指的就是这级缓冲：下游暂时不可用时，事件先留在本地，而不是丢掉已经读过的偏移。

### 安装

取与 Elasticsearch 相同主版本的 Linux 包，解压到采集机：

```bash
mkdir -p /opt/observe/beats
tar -xzf filebeat-7.17.9-linux-x86_64.tar.gz -C /opt/observe/beats
mv /opt/observe/beats/filebeat-7.17.9-linux-x86_64 /opt/observe/beats/filebeat
```

### 先打到控制台

在 `filebeat` 目录写 `console-input.yml`，输入用标准输入，避免一上来就碰权限和路径：

```yaml
filebeat.inputs:
  - type: stdin
    enabled: true
setup.template.settings:
  index.number_of_shards: 1
output.console:
  pretty: true
```

```bash
./filebeat -e -c console-input.yml
```

`-e` 把自身日志打到终端。启动后输入一行 `shelf-ping`，终端里会出现一条 JSON。值得看的字段：

- `@timestamp`：采集时间
- `message`：原始行
- `input.type`：这里是 `stdin`
- `agent` 或 `beat`：采集器版本和主机名，具体键随 7.x 小版本略有差异
- `host.name`：主机名

能看到这条 JSON，说明配置被解析、发布管道是通的。

### 改读文件

另写 `file-input.yml`：

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/shelf/*.log
setup.template.settings:
  index.number_of_shards: 1
output.console:
  pretty: true
```

```bash
mkdir -p /var/log/shelf
echo "shelf-open" >> /var/log/shelf/a.log
./filebeat -e -c file-input.yml
```

日志里应出现正在收集的文件路径。再执行 `echo "shelf-next" >> /var/log/shelf/a.log`，新行会马上变成事件。Filebeat 不轮询整文件，它跟的是末尾新增的内容。

### 附加字段

原始行不够区分业务时，可以加标签和自定义字段：

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/shelf/*.log
    tags: ["storefront", "lab"]
    fields:
      origin: storefront
    fields_under_root: true
output.console:
  pretty: true
```

`tags` 进数组。`fields` 默认放在 `fields` 对象下面；`fields_under_root: true` 时提升到顶层，后面的 Logstash 条件 `[origin]` 才能直接命中。改完重启进程，再追加一行，确认新事件里带有 `origin`。

### 写入 Elasticsearch

把输出从控制台换成集群地址。分片数实验环境用 1 即可。

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/shelf/*.log
    tags: ["storefront", "lab"]
    fields:
      origin: storefront
    fields_under_root: false
setup.template.settings:
  index.number_of_shards: 1
output.elasticsearch:
  hosts: ["http://127.0.0.1:9200"]
```

启动后日志里应有成功连接的记录。再往 `a.log` 追加一行。集群里会出现 `filebeat-*` 索引，文档的 `message` 就是那一行。这一步仍然没有解析日志格式，只是换了出口。

### 两个角色

Filebeat 里和“读文件”相关的职责要分开记。

**harvester**

- 负责一个已经打开的文件，按行读取并交给输出
- 一个文件一个 harvester，文件描述符在读取期间保持打开
- 文件被改名或删除后，只要 harvester 还没关闭，它会继续读那个已被删除的 inode，磁盘空间也不会释放
- 默认等到 `close_inactive` 到期才关闭。日志切割后若空间不下降，先看是不是旧文件仍被占着

**输入（旧称 prospector）**

- 管理 harvester，按路径发现要读的文件
- 类型为 `log` 时，为每个匹配文件启动一个 harvester
- 早期版本把发现文件的组件叫 prospector，并区分 log 与 stdin；7.x 统一称为 input，思路相同

偏移量记在数据目录的 registry 里。下游中断时，Filebeat 停在最后一次成功发布的位置，恢复后继续，而不是从头再读。进程重启时用 registry 重建内存状态。删掉 registry 会导致已读内容被重新发送，下游要能容忍重复，或者接受短暂的重复文档。

registry 是按文件身份（通常是 inode 加设备号）而不是只按路径记的。日志切割如果把文件改名再新建一个同名文件，新文件会被当成新来源从头读，旧文件在关闭前仍由原来的 harvester 读完。复制日志来“补数据”时如果 inode 没变，偏移不会回到开头。排错时先看 registry 里的偏移是否已经等于文件大小，再判断是采集器没读到，还是下游把事件丢了。

多个路径写在同一个 input 里：

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/nginx/*.log
      - /var/log/shelf/*.log
```

每个 input 在自己的 goroutine 里跑。通配符按采集机的文件系统展开，不会去读远程目录。

### 命令行

```bash
./filebeat -e -c shelf-es.yml
./filebeat -e -c shelf-es.yml -d "publish"
```

- `-e`：自身日志写到标准错误，而不是只写文件
- `-c`：指定配置文件
- `-d`：打开某一类调试日志，`publish` 用来看事件有没有发出去

### 直接采集代理日志

不使用模块时，只能拿到原始行：

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/*.log
    tags: ["edge"]
setup.template.settings:
  index.number_of_shards: 1
output.elasticsearch:
  hosts: ["http://127.0.0.1:9200"]
```

索引里的 `message` 是整行访问日志。能检索全文，但不能按状态码聚合。格式固定的日志应交给模块或 Logstash。

### 模块

模块把常见软件的路径、解析管道和字段约定打包好。列出当前包里的模块：

```bash
./filebeat modules list
```

列表很长，默认大多处于 Disabled，其中包含 nginx、mysql、redis、system 等。启用和关闭：

```bash
./filebeat modules enable nginx
./filebeat modules disable nginx
```

启用后，`modules.d/nginx.yml` 会从 `.disabled` 后缀变成有效文件。打开它，把访问日志和错误日志指到本机路径：

```yaml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
```

主配置改为加载模块，而不是再写一份重复的 `filebeat.inputs`：

```yaml
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
output.elasticsearch:
  hosts: ["http://127.0.0.1:9200"]
```

```bash
./filebeat -e -c edge.yml
```

若启动报错，提示缺少 `ingest-geoip` 一类的 Elasticsearch 插件，说明模块的 ingest 管道要用到它。在每个 Elasticsearch 节点上安装对应插件后重启节点，再启动 Filebeat。7.x 后期发行包往往已经带上 geoip，报错以实际日志为准。

刷新几次代理页面，或向错误日志追加一行：

```bash
echo "edge lab line" >> /var/log/nginx/error.log
```

文档里应出现已经拆开的字段，例如方法、路径、状态码，而不再只有一整段 `message`。模块清单和每个模块的变量以该版本的参考手册为准，包内列表会随小版本增减。

## Metricbeat

Metricbeat 按固定周期向操作系统或应用拉取指标，写成文档写入 Elasticsearch，用来看趋势而不是翻某一行日志。

### 模块与指标集

它由两层名字组成：

- Module：采集对象，例如系统、Nginx、MySQL
- Metricset：这个对象上的一组指标，例如 cpu、memory、network

一个模块可以带多个指标集。Redis 模块会分别取信息、键空间等不同集合，周期到了各取一次。

### 安装与配置

发行包与 Filebeat 放在同一父目录，版本对齐。编辑 `metricbeat.yml`，让它加载 `modules.d` 里已经启用的模块，并写到本机集群：

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression
setup.kibana:
  host: "http://127.0.0.1:5601"
output.elasticsearch:
  hosts: ["http://127.0.0.1:9200"]
processors:
  - add_host_metadata: ~
```

默认启用的是 system 模块，对应文件是 `modules.d/system.yml`。先不要打开一长串自己不看的模块，每个模块都会按周期打目标端口。

### 启动

```bash
./metricbeat -e
```

集群中会出现 `metricbeat-*` 索引，文档带 `event.module: system`。看不到数据时，先确认输出地址，再确认运行用户有没有读取 `/proc` 的权限。

### system 模块

```yaml
- module: system
  period: 10s
  metricsets:
    - cpu
    - load
    - memory
    - network
    - process
    - process_summary
```

`period` 是采集间隔。间隔越短，索引增长越快。进程指标集在主机进程很多时文档体积明显变大，实验环境可以先只留 cpu、load 和 memory。

### 查看与启用模块

```bash
./metricbeat modules list
```

system 默认 Enabled。其余如 nginx、mysql、redis、kubernetes 在 Disabled 列。用法和 Filebeat 一样：`modules enable` 去掉文件名上的 disabled 后缀，再按需要改主机和路径。

### Nginx 指标

指标不是从访问日志推算的，而是读取代理的状态页。编译或安装时要带上 `http_stub_status_module`。用下面的命令确认参数里有它：

```bash
nginx -V
```

在配置里增加只对本机开放的 location：

```nginx
location /edge-status {
    stub_status;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

```bash
nginx -s reload
curl -s http://127.0.0.1/edge-status
```

返回的纯文本大致三块：

- Active connections：当前正在处理的连接
- 三个累加数：已经接受的连接、已处理的握手、总请求数。握手数减去连接数若长期增大，说明有连接在排队或被丢弃，需要结合错误日志看，而不是只看这三个数的绝对值
- Reading / Writing / Waiting：正在读请求头、正在写响应、以及已经处理完并等待下一次请求的空闲连接。开启长连接时，Waiting 会占掉 Active 里的一大部分

状态页不要暴露到公网。它没有业务数据，但会泄露流量规模。

### 打开 nginx 模块

```bash
./metricbeat modules enable nginx
```

编辑 `modules.d/nginx.yml`，主机和状态路径与上面的 location 一致：

```yaml
- module: nginx
  period: 10s
  metricsets:
    - stubstatus
  hosts: ["http://127.0.0.1"]
  server_status_path: "edge-status"
```

有的版本把路径变量写成 `server_status_path`，有的放在 metricset 的 `path` 下，以当前文件里的注释为准。然后重新启动 Metricbeat。索引里应出现 `event.module` 为 nginx、`metricset.name` 为 stubstatus 的文档，字段对应活动连接和累加计数。

更多模块的主机、认证项和指标集名称，查该版本 Metricbeat 的模块参考，不要沿用别的主版本的示例键名。

## 和日志管道的分工

Filebeat 解决“发生过的一行行事件”，Metricbeat 解决“某一时刻的数值”。代理排错时两样都要：状态码分布来自解析后的访问日志，连接堆积来自 stub_status。Kibana 里它们是不同的索引模式，时间字段都用 `@timestamp`，才能画在同一段时间轴上对照。
