# 在 GitHub 上缩小搜索范围

读优秀仓库和它的实现，比只看封装后的 API 更容易建立自己的判断。下面是常用的检索语法，示例都换成了与本笔记无关的关键词。

## 常用词含义

- watch：订阅仓库动态，有新的讨论或推送会提醒你
- fork：把别人的仓库复制一份到自己的账号下，方便改完再提合并请求
- star：给仓库做一个书签，方便以后找回
- clone：把远程历史下载到本机
- follow：关注某个作者，时间线上会出现对方的公开动作

## in 关键字限制搜索范围

- 句式：`关键词 in:name`、`in:description` 或 `in:readme`
- `netty in:name`：仓库名里带 netty
- `netty in:description`：简介里带 netty
- `netty in:readme`：说明文件正文里带 netty
- 可以并列：`netty in:name,readme` 表示名字或说明文件任一处命中即可

## stars 或 fork 数量关键字查找

- 句式：
  - `关键词 stars:>N` 或 `stars:>=N`
  - 闭区间写成 `stars:N..M`
- 例子
  - 星标不少于 8000 的 kafka 相关仓库：`kafka stars:>=8000`
  - fork 落在 200 到 900：`kafka forks:200..900`
- 叠在一起
  - `kafka stars:>2000 forks:300..1200`

## awesome 加强搜索

- 句式：`awesome 主题`。这类仓库通常是书单、工具单或学习路线的目录，而不是某一个框架本身。
- 想找整理好的消息队列资料，可以搜 `awesome mq` 或 `awesome kafka`。

## 高亮显示某行代码

- 单行：在 blob 地址末尾加 `#L行号`
  - `https://github.com/octocat/Hello-World/blob/master/README#L1`
- 连续多行：`#L起始-L结束`
  - `https://github.com/octocat/Hello-World/blob/master/README#L1-L4`

## 项目内搜索

- 打开仓库页面后按英文字母 `t`，会弹出文件名快速跳转，用来在大仓库里按路径定位文件。

```mermaid
flowchart LR
    Open[打开仓库主页] --> PressT[按下 t]
    PressT --> Type[输入文件名片段]
    Type --> Jump[回车跳到该文件]
```

## 搜索某个地区内的开发者

- `location:` 限定资料里填写的地区
- `language:` 限定主要使用的编程语言
- 例子：`location:hangzhou language:go`
