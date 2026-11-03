# 用 Python 把配置里的 IP 换成服务器外网地址

## 前言

用 Compose 批量部署时，前端配置里常常写着回环地址。每台机器都手工改一遍容易漏。下面用一段 Python 完成两件事：

- 问一个公网回显服务，拿到这台机器的出口地址
- 读配置文件，把其中出现的 IPv4 换成这个地址

脚本假设文件里需要替换的是同一台机器的地址，不会区分“哪个 IP 该留在 127.0.0.1”。文件里如果混有别人的地址，要先改选择逻辑。

## 获取外网 ip

Python 3 用标准库即可，不必再依赖 `urllib2`：

```python
from urllib.request import urlopen

outbound = urlopen("https://ifconfig.me/ip", timeout=5).read().decode().strip()
```

`https://ifconfig.me/ip` 返回一行纯文本地址。内网机器没有公网出口时，这里会拿到出口网关的地址，或者直接超时。拿到之后放在变量里，供后面的替换使用。

也可以先在 shell 里确认回显是否正常：

```bash
curl -4 -fsS https://ifconfig.me/ip
```

示意输出：

```text
203.0.113.48
```

## 读取配置文件

接着读前端环境文件，用正则找出一行里的 IPv4，再整文件替换。

```python
def rewrite(path, fresh):
    pattern = re.compile(r"(?:\d{1,3}\.){3}\d{1,3}")
    with open(path, encoding="utf-8") as handle:
        rows = handle.readlines()
    found = None
    for row in rows:
        hit = pattern.search(row)
        if hit:
            found = hit.group(0)
            print(path, found, "=>", fresh)
            break
    if not found or found == fresh:
        return
    updated = "".join(row.replace(found, fresh) for row in rows)
    with open(path, "w", encoding="utf-8") as handle:
        handle.write(updated)
```

`findall` 会得到列表，这里用 `search` 只取最先出现的一个地址，避免空列表还去取下标。替换时遍历全部行，所以同一地址在别的变量里也会一起换掉。写回用原文件路径，执行前应先复制备份。

## 完整代码

保存为 `swap_addr.py`：

```python
# -*- coding: utf-8 -*-
import re
from urllib.request import urlopen

outbound = urlopen("https://ifconfig.me/ip", timeout=5).read().decode().strip()

def rewrite(path, fresh):
    pattern = re.compile(r"(?:\d{1,3}\.){3}\d{1,3}")
    with open(path, encoding="utf-8") as handle:
        rows = handle.readlines()
    found = None
    for row in rows:
        hit = pattern.search(row)
        if hit:
            found = hit.group(0)
            print("%s %s => %s" % (path, found, fresh))
            break
    if not found or found == fresh:
        return
    text = "".join(row.replace(found, fresh) for row in rows)
    with open(path, "w", encoding="utf-8") as handle:
        handle.write(text)

rewrite("env/admin-web.env", outbound)
rewrite("env/public-web.env", outbound)
```

替换前，`env/admin-web.env` 类似：

```text
NODE_ENV=production
ADMIN_API=http://127.0.0.1:8607/admin
MEDIA_API=http://127.0.0.1:8607/media
SITE_API=http://127.0.0.1:8607/site
SEARCH_API=http://127.0.0.1:8607/search
FILE_API=http://127.0.0.1:8610/
WEB_URL=http://127.0.0.1:9528
INDEX_API=http://127.0.0.1:8983/solr
KIBANA_URL=http://127.0.0.1:5601
```

执行：

```bash
python3 swap_addr.py
```

示意：

```text
env/admin-web.env 127.0.0.1 => 203.0.113.48
env/public-web.env 127.0.0.1 => 203.0.113.48
```

再打开 `env/admin-web.env`：

```text
NODE_ENV=production
ADMIN_API=http://203.0.113.48:8607/admin
MEDIA_API=http://203.0.113.48:8607/media
SITE_API=http://203.0.113.48:8607/site
SEARCH_API=http://203.0.113.48:8607/search
FILE_API=http://203.0.113.48:8610/
WEB_URL=http://203.0.113.48:9528
INDEX_API=http://203.0.113.48:8983/solr
KIBANA_URL=http://203.0.113.48:5601
```

回环地址已经换成出口地址。端口和路径没有被正则碰到，因为模式只匹配四段数字。若某一行必须保留 `127.0.0.1`，不要用这段“碰见第一个就全换”的逻辑，改成按变量名替换。
