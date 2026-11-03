# 用 database/sql 操作 MySQL

## 先分清库的类型

常见的数据库有 SQLite、MySQL、SQL Server、PostgreSQL 和 Oracle。MySQL 与 PostgreSQL 都是主流的关系型系统：同一类实体放进一张表，行是记录，列是字段。表结构要满足的约束，通常用三大范式来检查，避免同一事实在多处存放导致更新不一致。

## MySQL 里会反复用到的概念

### SQL 的三类语句

- DDL 定义库、表、列这些结构。
- DML 对行做增加、删除、修改、查询。
- DCL 管理账号和权限。

应用代码里最多的是 DML。建表属于 DDL，上线脚本里执行，不要散落在每次请求的路径上。

### 存储引擎

MySQL 的存储引擎可以按表选择。最常比较的是 MyISAM 和 InnoDB。

MyISAM 查询路径短，只支持表锁，不支持事务，崩溃后恢复能力弱。InnoDB 支持事务、行锁和外键，崩溃恢复更完整，现在是默认引擎。新表不要再选 MyISAM，除非有明确的只读归档需求。

### 事务

把多条 SQL 当成一件事。转账要扣一方、加另一方，两步必须同进同退。

### ACID

- 原子性：要么全部完成，要么全部撤销，不留做到一半的状态。
- 一致性：约束在事务前后都成立，例如余额不为负。
- 隔离性：并发事务彼此看不到不该看到的中间结果。
- 持久性：提交之后，即使进程崩溃，结果也已经落在存储上。

### 索引

索引多半组织成 B+ 树。它决定查找能不能少扫行，也决定写的时候要多维护一棵树。命中与否要看条件是否从索引的最左列开始用，对列做函数运算常常导致用不上索引。

### 还会碰到的运维话题

分库分表、注入、慢查询、主从复制和读写分离。注入和事务在本节后面有对应代码。慢查询先打开慢日志，确认是缺索引还是返回了过多的行，再决定改语句还是改表。

## database/sql 扮演的角色

`database/sql` 只规定连接、查询、预处理和事务这些接口，不包含任何一种数据库的私有协议。至少要注册一个驱动，MySQL 常用 `github.com/go-sql-driver/mysql`。

`sql.DB` 自带连接池，可以被多个 goroutine 同时使用。它不是一条连接，而是一组可借出的连接。

### 初始化模块并拉取驱动

```bash
go mod init example.com/lab/sqldemo
go get github.com/go-sql-driver/mysql
```

模块模式下，源码默认进入模块缓存，而不是 `GOPATH/src`。导入驱动时在包路径前加下划线，只为了执行它的 `init` 从而把驱动名 `mysql` 注册进 `database/sql`。

### 打开、探活、查询

```go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

func openDB() error {
	dsn := "app:app@tcp(127.0.0.1:3306)/lab_sql?charset=utf8mb4&parseTime=true"
	var err error
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err
	}
	if err = db.Ping(); err != nil {
		return err
	}
	fmt.Println("database reachable")
	return nil
}

type member struct {
	id   int
	name string
	age  int
}

func queryAdults() {
	const q = "SELECT id, name, age FROM member WHERE id > ?"
	rows, err := db.Query(q, 0)
	if err != nil {
		fmt.Printf("query: %v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var m member
		if err := rows.Scan(&m.id, &m.name, &m.age); err != nil {
			fmt.Printf("scan: %v\n", err)
			return
		}
		fmt.Printf("id=%d name=%s age=%d\n", m.id, m.name, m.age)
	}
	if err := rows.Err(); err != nil {
		fmt.Printf("rows: %v\n", err)
	}
}

func main() {
	if err := openDB(); err != nil {
		fmt.Println("open:", err)
		return
	}
	defer db.Close()
	queryAdults()
}
```

`Open` 只检查参数，不一定立刻拨号。`Ping` 才真正借一条连接试一次。`Scan` 的参数必须是指针，顺序与 SELECT 列表一致。循环结束后再看 `rows.Err`，迭代过程中的错误不会自动从 `Next` 返回。

### SetMaxOpenConns

```go
func (db *DB) SetMaxOpenConns(n int)
```

它限制同时打开的连接数。`n <= 0` 表示不限制，这也是默认值。若 `n` 比当前允许的闲置数还小，闲置上限会被压到同样的值。不设上限时，流量尖峰可能把数据库的连接数打满。

连接是在 `Query` 或 `Exec` 时借出的。`rows` 必须 `Close`，`QueryRow` 必须 `Scan`，否则那条连接不会还回池子。只 `Scan` 而不 `Close`，多行结果集仍会占着连接。

### SetMaxIdleConns

```go
func (db *DB) SetMaxIdleConns(n int)
```

它决定池里最多保留多少条空闲连接。`n <= 0` 时不保留，每次都重新拨号。`n` 大于最大打开数时，会被自动降到最大打开数。空闲连接能省去频繁握手，但留太多会占住数据库侧的会话。

## 增删改查

### 建库建表

```sql
CREATE DATABASE lab_sql;
USE lab_sql;

CREATE TABLE member (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(20) DEFAULT '',
    age INT DEFAULT 0,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 查询用的结构体

```go
type member struct {
	id   int
	name string
	age  int
}
```

字段未导出没有关系，`Scan` 在同一个包里能写这些字段。换到别的包就要导出，或使用 `sqlx` 的标签。

### 单行

`QueryRow` 永远返回非 nil 的 `*Row`。语句错误、没有行，都推迟到 `Scan` 才报告。没有行时错误是 `sql.ErrNoRows`。

```go
func fetchOne() {
	const q = "SELECT id, name, age FROM member WHERE id = ?"
	var m member
	err := db.QueryRow(q, 1).Scan(&m.id, &m.name, &m.age)
	if err != nil {
		fmt.Printf("scan: %v\n", err)
		return
	}
	fmt.Printf("id=%d name=%s age=%d\n", m.id, m.name, m.age)
}
```

### 多行

`Query` 的参数按顺序替换语句里的问号。

```go
func fetchMany() {
	const q = "SELECT id, name, age FROM member WHERE id > ?"
	rows, err := db.Query(q, 0)
	if err != nil {
		fmt.Printf("query: %v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var m member
		if err := rows.Scan(&m.id, &m.name, &m.age); err != nil {
			fmt.Printf("scan: %v\n", err)
			return
		}
		fmt.Printf("id=%d name=%s age=%d\n", m.id, m.name, m.age)
	}
}
```

### 插入

插入、更新、删除都走 `Exec`。返回的 `Result` 能提供自增主键和影响行数，不是结果集。

```go
func insertOne() {
	const q = "INSERT INTO member(name, age) VALUES (?, ?)"
	res, err := db.Exec(q, "周衡", 31)
	if err != nil {
		fmt.Printf("insert: %v\n", err)
		return
	}
	id, err := res.LastInsertId()
	if err != nil {
		fmt.Printf("last id: %v\n", err)
		return
	}
	fmt.Printf("inserted %d\n", id)
}
```

### 更新

```go
func updateOne() {
	const q = "UPDATE member SET age = ? WHERE id = ?"
	res, err := db.Exec(q, 32, 2)
	if err != nil {
		fmt.Printf("update: %v\n", err)
		return
	}
	n, err := res.RowsAffected()
	if err != nil {
		fmt.Printf("affected: %v\n", err)
		return
	}
	fmt.Printf("updated %d\n", n)
}
```

### 删除

```go
func deleteOne() {
	const q = "DELETE FROM member WHERE id = ?"
	res, err := db.Exec(q, 2)
	if err != nil {
		fmt.Printf("delete: %v\n", err)
		return
	}
	n, _ := res.RowsAffected()
	fmt.Printf("deleted %d\n", n)
}
```

`RowsAffected` 为 0 通常表示条件没有命中，不一定是错误。调用方要自己决定这算不算失败。

## 预处理

### 普通执行多了哪一步

不使用预处理时，客户端先把值嵌进完整语句，再整句发给服务器执行。值如果来自用户，嵌进去的瞬间就有注入风险。

预处理把语句拆开：

1. 先把带问号的命令发给服务器，服务器编译并记下这份计划。
2. 再单独发送数据，由服务器替换占位符。
3. 服务器执行并把结果返回。

同一份计划可以反复绑定不同的数据。

### 为什么值得做

重复执行的语句只编译一次，后面每次少一次解析。更重要的是，数据永远不会变成 SQL 文本的一部分，注入无路可走。

### 在 Go 里准备

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

返回的 `*Stmt` 可以被多个 goroutine 使用。用完要 `Close`，否则会占着服务端的预处理句柄。

查询：

```go
func prepQuery() {
	const q = "SELECT id, name, age FROM member WHERE id > ?"
	stmt, err := db.Prepare(q)
	if err != nil {
		fmt.Printf("prepare: %v\n", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(0)
	if err != nil {
		fmt.Printf("query: %v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var m member
		if err := rows.Scan(&m.id, &m.name, &m.age); err != nil {
			fmt.Printf("scan: %v\n", err)
			return
		}
		fmt.Printf("id=%d name=%s age=%d\n", m.id, m.name, m.age)
	}
}
```

插入可以复用同一份计划：

```go
func prepInsert() {
	const q = "INSERT INTO member(name, age) VALUES (?, ?)"
	stmt, err := db.Prepare(q)
	if err != nil {
		fmt.Printf("prepare: %v\n", err)
		return
	}
	defer stmt.Close()
	if _, err = stmt.Exec("梁舟", 18); err != nil {
		fmt.Printf("insert: %v\n", err)
		return
	}
	if _, err = stmt.Exec("何安", 22); err != nil {
		fmt.Printf("insert: %v\n", err)
		return
	}
	fmt.Println("batch inserted")
}
```

`db.Query` 带参数时，驱动内部也会走预处理。显式 `Prepare` 的价值在于同一语句要执行许多次，避免反复准备。

### 拼接会造成注入

下面这个函数把名字直接嵌进语句，是反面例子，不要在业务里这么写。

```go
func unsafeByName(name string) {
	q := fmt.Sprintf("SELECT id, name, age FROM member WHERE name='%s'", name)
	fmt.Println("SQL", q)
	var m member
	err := db.QueryRow(q).Scan(&m.id, &m.name, &m.age)
	if err != nil {
		fmt.Printf("exec: %v\n", err)
		return
	}
	fmt.Printf("member=%#v\n", m)
}
```

这些输入会改变语句结构，而不是被当成名字：

```go
unsafeByName("xxx' OR 1=1#")
unsafeByName("xxx' UNION SELECT id, name, age FROM member#")
unsafeByName("xxx' AND (SELECT COUNT(*) FROM member) < 10#")
```

`#` 在 MySQL 里会注释掉后面的引号。正确写法是 `WHERE name = ?`，把用户文本当作参数。

不同数据库的占位符不同：

| 数据库 | 占位符 |
| --- | --- |
| MySQL | `?` |
| PostgreSQL | `$1`、`$2` |
| SQLite | `?` 或 `$1` |
| Oracle | `:name` |

换驱动时要连占位符一起换，不能只改导入路径。

## 事务

### 它是最小的业务单元

事务不可再拆：对应一次完整业务，里面却可能有多条 DML。账户划转就是两次更新。MySQL 里只有 InnoDB 这类引擎支持事务，MyISAM 上的 `BEGIN` 不会带来真正的回滚。

### ACID 再看细一点

| 特性 | 含义 |
| --- | --- |
| 原子性 | 中途出错就回滚到开始之前，像没执行过 |
| 一致性 | 事务前后完整性约束都成立，精度、外键和后续触发的动作都算在内 |
| 隔离性 | 并发事务交叉执行时仍不把数据写脏。常见级别从低到高有读未提交、读已提交、可重复读、串行化 |
| 持久性 | 提交后的修改不会因为进程崩溃而消失 |

InnoDB 默认隔离级别是可重复读。级别越高，锁越多，吞吐越低。

### 三个方法

```go
func (db *DB) Begin() (*Tx, error)
func (tx *Tx) Commit() error
func (tx *Tx) Rollback() error
```

`Begin` 之后，语句要打在返回的 `tx` 上。打在 `db` 上等于开了另一条连接，不在这个事务里。

### 两次更新同进同退

```go
func adjustAges() {
	tx, err := db.Begin()
	if err != nil {
		fmt.Printf("begin: %v\n", err)
		return
	}
	rollback := func() {
		if rbErr := tx.Rollback(); rbErr != nil {
			fmt.Printf("rollback: %v\n", rbErr)
		}
	}

	res1, err := tx.Exec("UPDATE member SET age = 30 WHERE id = ?", 2)
	if err != nil {
		rollback()
		fmt.Printf("first: %v\n", err)
		return
	}
	n1, err := res1.RowsAffected()
	if err != nil {
		rollback()
		return
	}
	res2, err := tx.Exec("UPDATE member SET age = 40 WHERE id = ?", 3)
	if err != nil {
		rollback()
		fmt.Printf("second: %v\n", err)
		return
	}
	n2, err := res2.RowsAffected()
	if err != nil {
		rollback()
		return
	}
	fmt.Println(n1, n2)
	if n1 == 1 && n2 == 1 {
		if err := tx.Commit(); err != nil {
			fmt.Printf("commit: %v\n", err)
			return
		}
		fmt.Println("committed")
		return
	}
	rollback()
	fmt.Println("rolled back")
}
```

任一条件没有正好改到一行，就整笔撤回，避免只改了一半。`Commit` 之后不要再 `Rollback`，那个事务已经结束。

更稳的写法是用命名返回值配合 `defer`：函数返回错误或发生 panic 时回滚，否则提交。这样新增一条语句时不容易漏掉某个失败分支。

## 练习

用 `net/http` 和 `database/sql` 做一个很小的站点：注册时把用户名和口令摘要写入 MySQL，登录时按用户名查出摘要再比对。口令不要明文入库，查询一律使用占位符。

下一节的 sqlx 能把行直接扫进结构体，并简化按名字绑定和 `IN` 查询。标准库先跑通之后再换，排错时更容易分清是 SQL 的问题还是映射的问题。
