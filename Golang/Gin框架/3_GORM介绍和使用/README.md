# GORM介绍和使用

## 参考

官方手册把连接、模型和增删改查都列了出来。下面按一条能跑通的路径整理：先理解映射，再连上数据库，最后看字段约定。手册适合当查表，这篇笔记适合第一次把 CRUD 串起来。

## 什么是ORM

对象关系映射，是把内存里的结构体和数据库里的表对应起来。

```mermaid
flowchart LR
  obj["结构体字段"] --> orm["映射层"]
  orm --> row["表中的一行"]
  row --> orm
  orm --> obj
```

写代码时操作的是 `User` 这种类型，库负责生成 `INSERT`、`SELECT` 等语句。一行记录对应一个结构体值，一列对应一个字段。有了这层对应，就不必在业务函数里手写每一条 SQL。

## ORM的优缺点

**优点：**

- 少写重复的 SQL，模型改了可以让迁移跟着变，迭代更快

**缺点：**

- 中间多一次翻译，极端热点上不如手写语句
- 复杂查询不容易用链式调用表达完整
- 长期只靠 ORM，自己组织 SQL 的能力会变生疏

简单的增删改查用它，报表和需要锁、窗口函数的查询仍应写 SQL。

## gorm介绍

GORM 是 Go 实现的 ORM，文档覆盖主流关系库。当前常见的模块路径是 `gorm.io/gorm`，驱动分开安装；早期教学代码使用 `github.com/jinzhu/gorm`，打开方式是 `gorm.Open("mysql", dsn)`。两套 API 不要混在同一个项目里。下面示例沿用 v1 的调用形式，便于对照旧项目。

## 安装

v1：

```bash
go get -u github.com/jinzhu/gorm
```

v2 则是 `go get gorm.io/gorm`，再按数据库另装驱动，例如 `gorm.io/driver/mysql`。

## 连接数据库

每种数据库需要自己的驱动。v1 把驱动放在 `dialects` 子包，用空白导入触发注册：

```go
import _ "github.com/jinzhu/gorm/dialects/mysql"
```

postgres、sqlite、mssql 同样有对应的空白导入，不导入就无法按方言名打开。

### 连接MySQL

```go
import (
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"
)

func main() {
	dsn := "app:secret@(127.0.0.1:3306)/appdb?charset=utf8mb4&parseTime=True&loc=Local"
	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
	defer db.Close()
}
```

DSN 里的 `parseTime=True` 让日期列扫进 `time.Time`，`loc=Local` 使用本机时区。账号口令不要写死在仓库里，运行时从环境变量读。

### 连接PostgreSQL

同样调用 `gorm.Open`，第一个参数改成 `postgres`，并导入 postgres 方言。连接串使用键值形式：

```go
import (
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/postgres"
)

func main() {
	db, err := gorm.Open("postgres", "host=127.0.0.1 port=5432 user=app dbname=app password=secret sslmode=disable")
	if err != nil {
		panic(err)
	}
	defer db.Close()
}
```

### 连接Sqlite3

文件路径就是数据文件。适合单机演示，并发写能力有限。

```go
import (
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/sqlite"
)

func main() {
	db, err := gorm.Open("sqlite3", "demo.db")
	if err != nil {
		panic(err)
	}
	defer db.Close()
}
```

### 连接SQL Server

导入 mssql 方言，连接串使用 URL 形式：

```go
import (
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mssql"
)

func main() {
	db, err := gorm.Open("mssql", "sqlserver://user:secret@localhost:1433?database=appdb")
	if err != nil {
		panic(err)
	}
	defer db.Close()
}
```

## GORM基本示例

下面以 MySQL 为例走通迁移和增删改查。开始前需要一个已经能连上的实例，库可以先手工建好。

### Docker快速创建MySQL实例

本机没有 MySQL 时，可以用容器起一个。下面把容器内的 3306 映射到本机 13306，根口令只用于本地练习：

```bash
docker run --name mysql-demo -p 13306:3306 -e MYSQL_ROOT_PASSWORD=demo-pass -d mysql:8.0
```

再用客户端容器连进去。口令与上一步相同：

```bash
docker run -it --rm mysql:8.0 mysql -h127.0.0.1 -P13306 --default-character-set=utf8mb4 -uroot -pdemo-pass
```

### 创建数据库

GORM 的自动迁移负责表，不负责创建库。先在客户端执行：

```mysql
CREATE DATABASE appdb DEFAULT CHARSET utf8mb4;
```

### GORM操作MySQL

连接 `appdb`，用结构体做一次创建、查询、更新和删除。

```go
package main

import (
	"fmt"

	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"
)

type Member struct {
	ID     uint
	Name   string
	Gender string
	Hobby  string
}

func main() {
	db, err := gorm.Open("mysql", "root:demo-pass@(127.0.0.1:13306)/appdb?charset=utf8mb4&parseTime=True&loc=Local")
	if err != nil {
		panic(err)
	}
	defer db.Close()

	db.AutoMigrate(&Member{})

	m1 := Member{Name: "lin", Gender: "m", Hobby: "swim"}
	m2 := Member{Name: "chen", Gender: "f", Hobby: "chess"}
	db.Create(&m1)
	db.Create(&m2)

	var first Member
	db.First(&first)
	fmt.Printf("%#v\n", first)

	var byHobby Member
	db.Where("hobby = ?", "chess").First(&byHobby)
	fmt.Printf("%#v\n", byHobby)

	db.Model(&first).Update("hobby", "run")
	db.Delete(&first)
}
```

`AutoMigrate` 会按结构体补列，不会删掉已经存在的列。`Create` 之后主键会被填回结构体。`First` 取第一条，`Where` 的条件用问号承接外部输入，不要把输入拼进 SQL 字符串。

## GORM Model定义

模型一般就是结构体，字段用基本类型或指针。实现了 `sql.Scanner` 和 `driver.Valuer` 的类型也能当字段，用来做自定义列。

### gorm.Model

库自带一个可嵌入的结构，包含主键和三个时间字段：

```go
type Model struct {
	ID        uint `gorm:"primary_key"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt *time.Time
}
```

嵌进自己的模型后，这四列会一并出现：

```go
type User struct {
	gorm.Model
	Name string
}
```

不需要软删除时可以不嵌入，自己声明主键：

```go
type User struct {
	ID   int
	Name string
}
```

### 模型定义示例

标签写在反引号里，用来覆盖默认的列类型和索引：

```go
type User struct {
	gorm.Model
	Name         string
	Age          sql.NullInt64
	Birthday     *time.Time
	Email        string  `gorm:"type:varchar(100);unique_index"`
	Role         string  `gorm:"size:255"`
	MemberNumber *string `gorm:"unique;not null"`
	Num          int     `gorm:"AUTO_INCREMENT"`
	Address      string  `gorm:"index:addr"`
	IgnoreMe     int     `gorm:"-"`
}
```

`sql.NullInt64` 能区分“数据库里的 NULL”和 0。`IgnoreMe` 因为标签是 `-`，不会成为列。

### 结构体标记（tags）

标签可选。不写时按默认规则生成列名和类型。

#### 支持的结构体标记（Struct tags）

| 标记 | 描述 |
| --- | --- |
| Column | 指定列名 |
| Type | 指定列的 SQL 类型 |
| Size | 指定长度，字符串默认常常是 255 |
| PRIMARY_KEY | 这一列作为主键 |
| UNIQUE | 列值唯一 |
| DEFAULT | 列的默认值 |
| PRECISION | 小数精度 |
| NOT NULL | 不允许 NULL |
| AUTO_INCREMENT | 自增 |
| INDEX | 建索引；同名索引会合成复合索引 |
| UNIQUE_INDEX | 与 INDEX 类似，但是唯一索引 |
| EMBEDDED | 把结构体嵌入为列的一组 |
| EMBEDDED_PREFIX | 给嵌入字段的列名加前缀 |
| - | 忽略该字段 |

#### 关联相关标记（tags）

| 标记 | 描述 |
| --- | --- |
| MANY2MANY | 多对多时的中间表 |
| FOREIGNKEY | 外键字段 |
| ASSOCIATION_FOREIGNKEY | 关联另一侧的键 |
| POLYMORPHIC | 多态关联的类型列 |
| POLYMORPHIC_VALUE | 多态类型的取值 |
| JOINTABLE_FOREIGNKEY | 中间表里指向本模型的外键 |
| ASSOCIATION_JOINTABLE_FOREIGNKEY | 中间表里指向对方的外键 |
| SAVE_ASSOCIATIONS | 保存时是否连带保存关联 |
| ASSOCIATION_AUTOUPDATE | 更新时是否自动处理关联 |
| ASSOCIATION_AUTOCREATE | 创建时是否自动处理关联 |
| ASSOCIATION_SAVE_REFERENCE | 是否自动保存引用关系 |
| PRELOAD | 是否自动预加载关联 |

关联标签只在声明了关系字段时才有意义。初学阶段先把单表字段写对，再加外键。

## 主键、表名、列名的约定

### 主键（Primary Key）

默认把名为 `ID` 的字段当作主键。

```go
type User struct {
	ID   string
	Name string
}

type Animal struct {
	AnimalID int64 `gorm:"primary_key"`
	Name     string
	Age      int64
}
```

第二种没有 `ID`，必须用标签指出主键，否则迁移和更新无法定位行。

### 表名（Table Name）

默认表名是结构体名的复数蛇形，`User` 对应 `users`。

```go
type User struct{}

func (User) TableName() string {
	return "profiles"
}
```

方法也可以看字段再决定表名，例如管理员落在另一张表。这种动态表名容易让迁移和查询不一致，只在确实要分表时使用。

关闭复数规则后，`User` 的默认表名变成 `user`：

```go
db.SingularTable(true)
```

单次查询也可以不改模型，直接指定表：

```go
db.Table("archived_users").CreateTable(&User{})

var archived []User
db.Table("archived_users").Find(&archived)

db.Table("archived_users").Where("name = ?", "lee").Delete(&User{})
```

全局改前缀：

```go
gorm.DefaultTableNameHandler = func(db *gorm.DB, defaultTableName string) string {
	return "app_" + defaultTableName
}
```

这样所有默认表名都会带上 `app_`。已经实现 `TableName` 的模型是否再经过这个函数，要看所用版本，升级时用一张表实际看一眼。

### 列名（Column Name）

字段名会转成蛇形小写作为列名。

```go
type User struct {
	ID        uint
	Name      string
	Birthday  time.Time
	CreatedAt time.Time
}
```

对应列是 `id`、`name`、`birthday`、`created_at`。要改用别的列名时写 `column` 标签：

```go
type Animal struct {
	AnimalId int64     `gorm:"column:beast_id"`
	Birthday time.Time `gorm:"column:day_of_the_beast"`
	Age      int64     `gorm:"column:age_of_the_beast"`
}
```

标签里的名字必须和真实列一致，否则查询会报未知列。

### 时间戳跟踪

#### CreatedAt

模型含有 `CreatedAt` 时，`Create` 会填上当前时间。

```go
db.Create(&user)

db.Model(&user).Update("CreatedAt", time.Now())
```

一般不要手工改创建时间，除非是在导入历史数据。

#### UpdatedAt

含有 `UpdatedAt` 时，`Save` 和字段更新都会把它刷新成当前时间。

```go
db.Save(&user)

db.Model(&user).Update("name", "lee")
```

只想改某一列、又不想动更新时间时，需要看版本是否提供跳过钩子的更新方式，默认路径会维护这个字段。

#### DeletedAt

模型含有 `DeletedAt` 时，`Delete` 不会执行物理删除，而是把该字段写成当前时间。之后的普通查询会自动加上“删除时间为空”的条件，因此这条记录像是消失了，行仍留在表里。要真正删掉或把已删除的行查出来，需要额外指定非作用域查询。这个字段用指针，零值 `nil` 表示尚未删除。
