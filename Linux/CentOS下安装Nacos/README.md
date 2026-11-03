# CentOS 下安装 Nacos

## 前言

服务一多，注册和配置如果还写在每个进程里，换地址就要重新打包。Nacos 把注册表和配置中心放到单独的进程。下面在 CentOS 上用单机模式跑起来，并把数据从内嵌库改到 MySQL，方便以后整体迁走。

## 下载 Linux 版 Nacos

发布页：https://github.com/alibaba/nacos/releases

选服务器包，例如 2.3.2：

```bash
cd /usr/local/src
wget https://github.com/alibaba/nacos/releases/download/2.3.2/nacos-server-2.3.2.tar.gz
tar -xzf nacos-server-2.3.2.tar.gz
cd nacos/bin
```

单机演示不必给 JVM 很大的堆。编辑 `startup.sh`，找到单机分支，把内存改成机器能承受的值：

```bash
vi startup.sh
```

```bash
if [[ "${MODE}" == "standalone" ]]; then
    JAVA_OPT="${JAVA_OPT} -Xms256m -Xmx384m -Xmn128m"
    JAVA_OPT="${JAVA_OPT} -Dnacos.standalone=true"
else
    JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn192m"
    JAVA_OPT="${JAVA_OPT} -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m"
fi
```

数字只作用于这一次安装。内存更紧时再往下调，但堆太小会在导入大量配置时触发频繁回收。

## Nacos + MySQL 配置持久化

默认数据在内嵌的 Derby 里，目录一拷走就容易和版本缠在一起。要迁到别的机器，改成 MySQL 更直接。

在库里执行 `nacos/conf/mysql-schema.sql`（旧包里叫 `nacos-mysql.sql`）。先建库再导入：

```sql
CREATE DATABASE registry_meta DEFAULT CHARACTER SET utf8mb4;
USE registry_meta;
SOURCE /usr/local/src/nacos/conf/mysql-schema.sql;
```

### 导入遇到问题

MySQL 5.6 或没打开大索引前缀时，可能在建表中途停下，报错类似：

```text
Specified key was too long; max key length is 767 bytes
```

先看索引前缀开关：

```sql
SHOW VARIABLES LIKE 'innodb_large_prefix';
```

结果是 OFF 时：

```sql
SET GLOBAL innodb_large_prefix = ON;
```

接着看文件格式。要使用长索引，格式得是 Barracuda，而行格式通常还要是 DYNAMIC：

```sql
SHOW VARIABLES LIKE 'innodb_file_format';
SHOW VARIABLES LIKE 'innodb_default_row_format';
```

不是 Barracuda 时：

```sql
SET GLOBAL innodb_file_format = 'Barracuda';
SET GLOBAL innodb_default_row_format = DYNAMIC;
```

然后重新导入。MySQL 8.0 默认已经允许更长的索引，一般不会碰到这条错误；若仍失败，优先检查库的字符集是不是 utf8mb4，以及用的是不是与 Nacos 版本匹配的那份 SQL。

### 修改 nacos 配置

```bash
vi /usr/local/src/nacos/conf/application.properties
```

在文件末尾加上数据源。口令换成实际库账号：

```properties
spring.sql.init.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/registry_meta?characterEncoding=utf8&connectTimeout=2000&socketTimeout=5000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=registry
db.password.0=reg-pass-3308
```

2.2 之后的包认 `db.user.0` 这种带序号的键。只写 `db.user` 时，进程能起来，但连不上库，日志里会报数据源为空。

回到 `bin`，用单机模式启动：

```bash
cd /usr/local/src/nacos/bin
bash ./startup.sh -m standalone
```

控制台：

```text
http://10.8.20.18:8848/nacos
```

初始用户名和口令都是 `nacos`。进去之后立刻在用户管理里改掉。页面能打开配置列表，就说明这台 CentOS 上的单机实例已经可用。集群模式要改 `cluster.conf` 并去掉 `-m standalone`，那是另一步，不在这次安装里。
