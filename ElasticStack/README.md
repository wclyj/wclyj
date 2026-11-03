# Elastic Stack 笔记

ELK 原来指 Elasticsearch、Logstash、Kibana 三件套。采集代理 Beats 加入之后，这一组工具更常被叫做 Elastic Stack。Elasticsearch 负责存储和检索，Beats 负责在数据源旁边做轻量采集，Logstash 负责解析和转换，Kibana 负责查询和画图。

下面几篇按这个顺序展开，示例里的索引名、路径和业务字段都换成了独立的一套，阅读时不要和某一套现成安装脚本逐行对照。

## 目录

- [Elasticsearch 介绍与安装](./1_ElasticSearch介绍与安装/README.md)
- [Beats 入门](./2_Beats入门简介/README.md)
- [Kibana 安装与界面](./3_Kibana安装与介绍/README.md)
- [Logstash 入门](./4_Logstash入门简介/README.md)
- [综合案例：行为日志管道](./5_ElasticStack综合案例/README.md)
- [多模块站点的日志收集](./6_使用ELK搭建蘑菇博客日志收集/README.md)

从 5.0 起这套组件使用统一的版本号。笔记里的命令按 7.x 的单机实验来写。
