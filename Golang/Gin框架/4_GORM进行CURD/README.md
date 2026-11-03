# GORM 的增删改查

## 来源

步骤整理自公开的 GORM 用法说明，原文链接不重复。下文示例基于 jinzhu 维护的 v1 风格 API：`*gorm.DB`，打开连接用 `gorm.Open("mysql", dsn)`。v2 的导入路径和部分方法名已改，对照时先看自己项目锁的主版本。

## 介绍

CRUD 就是创建、读取、更新、删除。下面按这四类说明 GORM 怎么生成 SQL。

# CRUD

文中的 `db` 都是已经连上的 `*gorm.DB`：

```go
import "github.com/jinzhu/gorm"

// 另起一行做空白导入，路径是 github.com/jinzhu/gorm/dialects 下的 mysql 包，用来注册驱动。

func open() *gorm.DB {
    dsn := "app:secret@tcp(127.0.0.1:3306)/shop?charset=utf8mb4&parseTime=True&loc=Local"
    conn, err := gorm.Open("mysql", dsn)
    if err != nil {
        panic(err)
    }
    return conn
}
```

用完调用 `Close`。后面的链式调用都从这份 `conn` 开始。

## 创建

### 创建记录

模型先用结构体描述：

```go
type Member struct {
    ID   int64
    Nick string
    Years int64
}
```

`NewRecord` 看主键还是不是零值。为零说明库里还没有这条，接着 `Create` 插入。插入成功后主键会被回填，再次 `NewRecord` 得到假。

```go
m := Member{Nick: "lina", Years: 27}
fresh := db.NewRecord(m)
db.Create(&m)
stored := db.NewRecord(m)
_, _ = fresh, stored
```

`Create` 必须传指针，否则主键填不回调用方的变量。

### 默认值

字段上的 tag 可以写数据库默认值：

```go
type Member struct {
    ID   int64
    Nick string `gorm:"default:'访客'"`
    Years int64
}
```

创建时，零值字段不会出现在 INSERT 列表里。语句执行完，GORM 再把数据库填上的默认值读回来。

```go
m := Member{Nick: "", Years: 27}
db.Create(&m)
```

实际接近 `INSERT INTO members(years) VALUES (27)`。`nick` 留给表定义，结果是“访客”。

`0`、`""`、`false` 以及其他零值都走这条规则。若零值本身是合法业务数据，改用指针，或改用实现了 `Scanner` 与 `Valuer` 的类型。

#### 用指针字段保存零值

```go
type Member struct {
    ID   int64
    Nick *string `gorm:"default:'访客'"`
    Years int64
}
blank := ""
m := Member{Nick: &blank, Years: 27}
db.Create(&m)
```

指针非 nil，即使指向空串，也会写入列，库里的 nick 是空串而不是默认的“访客”。

#### 借助 Scanner/Valuer 写入零值

```go
type Member struct {
    ID   int64
    Nick sql.NullString `gorm:"default:'访客'"`
    Years int64
}
m := Member{Nick: sql.NullString{String: "", Valid: true}, Years: 27}
db.Create(&m)
```

`Valid: true` 表示“这是一个有意义的空串”。`Valid: false` 才会被当成 NULL。

### 扩展创建选项

个别数据库允许在 INSERT 后追加冲突处理。v1 用 `Set` 把片段接到语句末尾。PostgreSQL 可以这样表达“冲突则更新”：

```go
db.Set("gorm:insert_option", "ON CONFLICT (sku) DO UPDATE SET title = EXCLUDED.title").Create(&goods)
```

生成的语句在普通 INSERT 之后带上这段 `ON CONFLICT`。不同方言的语法不要混用。

## 查询

### 一般查询

```go
db.First(&member)          // 按主键升序取第一条
db.Take(&member)           // 不保证顺序，取一条
db.Last(&member)           // 按主键降序取一条
db.Find(&members)          // 全部
db.First(&member, 15)      // 整型主键等于 15
```

`First`、`Last` 在找不到记录时会返回 `ErrRecordNotFound`。`Find` 找不到时得到空切片，不算错误。

### Where 条件

#### 普通SQL查询

条件里的问号是占位符，不要自己拼接字符串。

```go
db.Where("nick = ?", "lina").First(&member)
db.Where("nick = ?", "lina").Find(&members)
db.Where("nick <> ?", "lina").Find(&members)
db.Where("nick IN (?)", []string{"lina", "owen"}).Find(&members)
db.Where("nick LIKE ?", "li%").Find(&members)
db.Where("nick = ? AND years >= ?", "lina", 22).Find(&members)
db.Where("updated_at > ?", weekAgo).Find(&members)
db.Where("created_at BETWEEN ? AND ?", weekAgo, today).Find(&members)
```

`LIKE` 的百分号写在 Go 字符串里，不要写进 SQL 骨架，否则无法绑定。

#### Struct & Map查询

结构体和映射都会被展开成等值条件：

```go
db.Where(&Member{Nick: "lina", Years: 20}).First(&member)
db.Where(map[string]any{"nick": "lina", "years": 20}).Find(&members)
db.Where([]int64{20, 21, 22}).Find(&members)
```

主键切片会变成 `id IN (...)`。

用结构体当条件时，零值字段会被丢掉：

```go
db.Where(&Member{Nick: "lina", Years: 0}).Find(&members)
// 只有 nick，years 不参与
```

要查询“年龄确实是 0”，把字段改成指针或 `sql.NullInt64`：

```go
type Member struct {
    gorm.Model
    Nick  string
    Years *int
}
```

映射不受这条限制，`map[string]any{"years": 0}` 会生成 `years = 0`。

### Not 条件

`Not` 是把条件取反，参数形式和 `Where` 相同。

```go
db.Not("nick", "lina").First(&member)
db.Not("nick", []string{"lina", "owen"}).Find(&members)
db.Not([]int64{1, 2, 3}).First(&member)
db.Not([]int64{}).First(&member) // 空切片不会变成非法的 NOT IN ()
db.Not("nick = ?", "lina").First(&member)
db.Not(Member{Nick: "lina"}).First(&member)
```

### Or条件

`Or` 接在已有条件后面，两侧用 OR 连接。

```go
db.Where("role = ?", "admin").Or("role = ?", "owner").Find(&members)
db.Where("nick = ?", "lina").Or(Member{Nick: "owen"}).Find(&members)
db.Where("nick = ?", "lina").Or(map[string]any{"nick": "owen"}).Find(&members)
```

多个 `Or` 是链式的。若还要和另一组 AND 组合，先用括号写进一条 SQL 字符串，避免优先级和你想的不一样。

### 内联条件

`First`、`Find` 等方法的额外参数相当于一次 `Where`，但只对紧挨着的那一次查询生效，不会留给后面的链式调用。

```go
db.First(&member, 23)
db.First(&member, "id = ?", "sku-23")
db.Find(&members, "nick = ?", "lina")
db.Find(&members, "nick <> ? AND years > ?", "lina", 20)
db.Find(&members, Member{Years: 20})
db.Find(&members, map[string]any{"years": 20})
```

所以 `db.Find(&rows, "id = ?", 1).Count(&n)` 里，`Count` 看不到那个内联条件。

### 额外查询选项

`Set("gorm:query_option", ...)` 把片段追加到 SELECT 末尾，例如加锁读：

```go
db.Set("gorm:query_option", "FOR UPDATE").First(&member, 10)
```

要包在事务里才有意义，否则锁马上随语句结束释放。

### FirstOrInit

按结构体或映射查找第一条。没有就在内存里初始化一个对象，不写数据库。

```go
db.FirstOrInit(&member, Member{Nick: "nobody"})
// 没有记录时 member.Nick == "nobody"，ID 仍是零

db.Where(Member{Nick: "Lina"}).FirstOrInit(&member)
db.FirstOrInit(&member, map[string]any{"nick": "lina"})
```

#### Attrs

`Attrs` 只在“没找到”时填到对象上。找到了就忽略这些属性。

```go
db.Where(Member{Nick: "nobody"}).Attrs(Member{Years: 20}).FirstOrInit(&member)
db.Where(Member{Nick: "nobody"}).Attrs("years", 20).FirstOrInit(&member)
db.Where(Member{Nick: "Lina"}).Attrs(Member{Years: 30}).FirstOrInit(&member)
// 找到时 Years 仍是库里的旧值
```

#### Assign

`Assign` 无论是否找到，都把属性写进对象，但 `FirstOrInit` 仍然不落库。

```go
db.Where(Member{Nick: "nobody"}).Assign(Member{Years: 20}).FirstOrInit(&member)
db.Where(Member{Nick: "Lina"}).Assign(Member{Years: 30}).FirstOrInit(&member)
// 找到时对象里的 Years 变成 30，数据库还没更新
```

### FirstOrCreate

和 `FirstOrInit` 类似，但找不到时会 `INSERT`。

```go
db.FirstOrCreate(&member, Member{Nick: "nobody"})
db.Where(Member{Nick: "Lina"}).FirstOrCreate(&member)
```

#### Attrs

找不到时，`Attrs` 里的字段会一起插入。找到时不覆盖已有列。

```go
db.Where(Member{Nick: "nobody"}).Attrs(Member{Years: 20}).FirstOrCreate(&member)
db.Where(Member{Nick: "lina"}).Attrs(Member{Years: 30}).FirstOrCreate(&member)
```

#### Assign

无论找没找到，`Assign` 的值都会写进库：没有就插入，已有就更新。

```go
db.Where(Member{Nick: "nobody"}).Assign(Member{Years: 20}).FirstOrCreate(&member)
db.Where(Member{Nick: "lina"}).Assign(Member{Years: 30}).FirstOrCreate(&member)
```

第二条会生成针对该主键的 `UPDATE`，把 years 改成 30。

### 高级查询

#### 子查询

v1 用 `SubQuery` 得到一段可嵌入的表达式：

```go
avg := db.Table("bills").Select("AVG(amount)").Where("state = ?", "paid").SubQuery()
db.Where("amount > ?", avg).Find(&bills)
```

生成的 SQL 在 WHERE 里嵌一个 `SELECT AVG(amount) ...`。v2 改为 `db.Table(...).Select(...)` 直接当参数，不再调用 `SubQuery`。

#### 选择字段

`Select` 缩小列清单。不写就相当于 `SELECT *`。

```go
db.Select("nick, years").Find(&members)
db.Select([]string{"nick", "years"}).Find(&members)
db.Table("members").Select("COALESCE(years, ?)", 42).Rows()
```

只要部分列时，结构体里其余字段保持零值，保存回去可能把它们覆盖成零。读出来只为展示时问题不大。

#### 排序

`Order` 追加 `ORDER BY`。第二个参数 `reorder` 为真时，丢掉前面已经写过的排序。

```go
db.Order("years desc, nick").Find(&members)
db.Order("years desc").Order("nick").Find(&members)
db.Order("years desc").Find(&older).Order("years", true).Find(&byYears)
```

第三次查询只剩 `ORDER BY years`，不再是降序。

#### 数量

`Limit` 加上 `LIMIT`。传入 `-1` 表示取消之前的限制，方便复用链式对象。

```go
db.Limit(3).Find(&members)
db.Limit(10).Find(&page).Limit(-1).Find(&all)
```

#### 偏移

`Offset` 跳过前面若干行，常和 `Limit` 一起做翻页。`-1` 同样表示取消。

```go
db.Offset(3).Find(&members)
db.Limit(10).Offset(20).Find(&page)
db.Offset(10).Find(&skipped).Offset(-1).Find(&all)
```

大偏移会让数据库仍然扫描被跳过的行。页码很深时改用“记住上一页最后的主键”来续查。

#### 总数

`Count` 把前面的查询改成 `COUNT`。它会覆盖 `SELECT` 列表，所以要放在链式调用的最后。若 `Select` 里已经写了 `count(...)`，则保留你的表达式。

```go
db.Where("nick = ?", "lina").Or("nick = ?", "owen").Find(&members).Count(&total)
db.Model(&Member{}).Where("nick = ?", "lina").Count(&total)
db.Table("archived_members").Count(&total)
db.Table("archived_members").Select("count(distinct nick)").Count(&total)
```

先 `Find` 再 `Count` 会发两条 SQL：一条取行，一条计数。只要总数时不要先 `Find`。

#### Group & Having

分组后既可以用 `Rows` 逐行扫描，也可以 `Scan` 进切片。

```go
type DaySum struct {
    Day   time.Time
    Total int64
}

var days []DaySum
db.Table("bills").
    Select("date(created_at) as day, sum(amount) as total").
    Group("date(created_at)").
    Scan(&days)

db.Table("bills").
    Select("date(created_at) as day, sum(amount) as total").
    Group("date(created_at)").
    Having("sum(amount) > ?", 100).
    Scan(&days)
```

`Scan` 按列名或别名匹配结构体字段，别名要和字段能对上，例如 `day` 对 `Day`。

#### 连接

`Joins` 写入 JOIN 片段，占位符仍然可用。

```go
db.Table("members").
    Select("members.nick, inboxes.addr").
    Joins("left join inboxes on inboxes.member_id = members.id").
    Scan(&results)

db.Joins("JOIN inboxes ON inboxes.member_id = members.id AND inboxes.addr = ?", "lina@example.com").
    Joins("JOIN cards ON cards.member_id = members.id").
    Where("cards.pan = ?", "411111111111").
    Find(&member)
```

左连接保留没有邮箱的会员；内连接会把他们滤掉。

### Pluck

只要一列、并且想直接得到切片时用 `Pluck`。多列请改 `Scan` 或 `Find`。

```go
var years []int64
db.Model(&Member{}).Pluck("years", &years)

var nicks []string
db.Table("archived_members").Pluck("nick", &nicks)

db.Select("nick, years").Find(&members)
```

`Pluck` 前若先 `Find` 了整行，会多一次没有必要的查询。直接从 `Model` 或 `Table` 开始即可。

### 扫描

`Scan` 把结果写入任意结构体，不要求它是模型。

```go
type Brief struct {
    Nick  string
    Years int
}

var one Brief
db.Table("members").Select("nick, years").Where("nick = ?", "Antonio").Scan(&one)

var many []Brief
db.Table("members").Select("nick, years").Where("id > ?", 0).Scan(&many)

db.Raw("SELECT nick, years FROM members WHERE nick = ?", "Antonio").Scan(&one)
```

`Raw` 适合数据库特有的语法。能用链式 API 表达的查询不必先写成整段 SQL。

## 链式操作相关

### 链式操作

在调用会真正发 SQL 的方法之前，中间步骤只是在累积条件。因此可以把公共过滤写成函数，返回 `*gorm.DB`。

```go
tx := db.Where("nick = ?", "lina")
if young {
    tx = tx.Where("years = ?", 20)
} else {
    tx = tx.Where("years = ?", 30)
}
if activeOnly {
    tx = tx.Where("active = ?", true)
}
```

### 立即执行方法

立刻发往数据库的方法包括 `Create`、`First`、`Find`、`Take`、`Save`、`Update`、`Updates`、`Delete`、`Scan`、`Row`、`Rows`。

```go
tx.Find(&member)
```

若前面三个条件都加上，语句类似：

```sql
SELECT * FROM members WHERE nick = 'lina' AND years = 30 AND active = 1;
```

### 范围

`Scopes` 把“返回 `*gorm.DB` 的函数”接进链条，用来复用过滤。

```go
func amountOver(db *gorm.DB) *gorm.DB {
    return db.Where("amount >= ?", 1500)
}
func paidByCard(db *gorm.DB) *gorm.DB {
    return db.Where("pay_code = ?", "CARD")
}
func paidByCod(db *gorm.DB) *gorm.DB {
    return db.Where("pay_code = ?", "COD")
}
func withStatus(states []string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Scopes(amountOver).Where("status IN (?)", states)
    }
}

db.Scopes(amountOver, paidByCard).Find(&bills)
db.Scopes(amountOver, paidByCod).Find(&bills)
db.Scopes(amountOver, withStatus([]string{"paid", "shipped"})).Find(&bills)
```

信用卡和货到付款必须使用不同的 `pay_code`，否则两个 Scope 会滤出同一批订单。

### 多个立即执行方法

连续调用多个立即执行方法时，后一次会沿用前一次的普通 `Where`，但不沿用内联条件。

```go
db.Where("nick LIKE ?", "lina%").Find(&members, "id IN (?)", []int{1, 2, 3}).Count(&total)
```

第一条 SQL 同时有 `LIKE` 和 `id IN`。`Count` 只保留 `LIKE`，内联的 `id IN` 不会出现。需要同一组条件时，把它们都写成 `Where`，不要塞进 `Find` 的参数。

## 更新

### 更新所有字段

`Save` 按主键更新模型的全部列，包括你没改过的零值，也会刷新 `updated_at`。

```go
db.First(&member)
member.Nick = "七米"
member.Years = 41
db.Save(&member)
```

主键为空时 `Save` 可能变成插入。更新前要确认 `ID` 已经有值。

### 更新修改字段

只改指定列用 `Update` 或 `Updates`。

```go
db.Model(&member).Update("nick", "hello")
db.Model(&member).Where("active = ?", true).Update("nick", "hello")
db.Model(&member).Updates(map[string]any{"nick": "hello", "years": 18, "active": false})
db.Model(&member).Updates(Member{Nick: "hello", Years: 18})
```

映射会写入给出的每一个键，因此 `false` 和 `0` 也能更新。结构体仍然跳过零值，所以下面这句不会产生有效赋值：

```go
db.Model(&member).Updates(Member{Nick: "", Years: 0, Active: false})
```

要把字符串改成空、把布尔改成假，用映射。

### 更新选定字段

`Select` 限定允许更新的列，`Omit` 排除列。两者都配合 `Updates`。

```go
db.Model(&member).Select("nick").Updates(map[string]any{"nick": "hello", "years": 18, "active": false})
db.Model(&member).Omit("nick").Updates(map[string]any{"nick": "hello", "years": 18, "active": false})
```

第一句只动 nick（以及自动维护的时间戳）。第二句动 years 和 active，不动 nick。

### 无Hooks更新

普通更新会跑 `BeforeUpdate`、`AfterUpdate`，刷新 `UpdatedAt`，并可能保存关联。要跳过这些，用 `UpdateColumn` 与 `UpdateColumns`。

```go
db.Model(&member).UpdateColumn("nick", "hello")
db.Model(&member).UpdateColumns(Member{Nick: "hello", Years: 18})
```

生成的 UPDATE 不含 `updated_at`。审计字段不能漏时不要用这对方法。

### 批量更新

不带具体主键、按条件更新多行时，钩子不会逐行执行。

```go
db.Table("members").Where("id IN (?)", []int{10, 11}).Updates(map[string]any{"nick": "hello", "years": 18})
db.Model(Member{}).Updates(Member{Nick: "hello", Years: 18})
affected := db.Model(Member{}).Updates(Member{Nick: "hello", Years: 18}).RowsAffected
_ = affected
```

`Model(Member{})` 没有 WHERE 时会更新整张表。批量改数据前先把条件写上，并在事务里核对 `RowsAffected`。

### 使用SQL表达式更新

列的新值若依赖旧值，用 `gorm.Expr`，避免先查到内存再写回造成丢失更新。

```go
var member Member
db.First(&member)
db.Model(&member).Update("years", gorm.Expr("years * ? + ?", 2, 100))
db.Model(&member).Updates(map[string]any{"years": gorm.Expr("years * ? + ?", 2, 100)})
db.Model(&member).UpdateColumn("years", gorm.Expr("years - ?", 1))
db.Model(&member).Where("years > ?", 10).UpdateColumn("years", gorm.Expr("years - ?", 1))
```

表达式里的表列名用数据库列，不是 Go 字段名。

### 修改Hooks中的值

在 `BeforeSave` 或 `BeforeUpdate` 里改最终写入的列，调用 `scope.SetColumn`，不要只改结构体字段却指望钩子之后的语句一定看得到。

```go
func (m *Member) BeforeSave(scope *gorm.Scope) error {
    if len(m.Secret) == 0 {
        return nil
    }
    hashed, err := bcrypt.GenerateFromPassword([]byte(m.Secret), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    return scope.SetColumn("secret_hash", hashed)
}
```

明文口令字段不要放进默认的 `Save` 列清单，可以用 `json:"-"` 加 `gorm:"-"` 把它排除出持久化。

### 其它更新选项

和查询类似，`gorm:update_option` 把额外片段接到 UPDATE 后面：

```go
db.Model(&member).Set("gorm:update_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Update("nick", "hello")
```

这是 SQL Server 的计划提示。MySQL 上不要原样复制。

## 删除

### 删除记录

`Delete` 靠主键定位。主键是零值时，可能变成删除该模型的全部行。调用前检查主键。

```go
db.Delete(&inbox)
db.Set("gorm:delete_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Delete(&inbox)
```

### 批量删除

按条件删除时传入空模型，条件写在 `Where` 或 `Delete` 的额外参数里。

```go
db.Where("addr LIKE ?", "%lina%").Delete(Inbox{})
db.Delete(Inbox{}, "addr LIKE ?", "%lina%")
```

两条都会删掉匹配的邮箱行。生产环境先用同样的 `Where` 做一次 `Count`。

### 软删除

模型若嵌入 `gorm.Model`，或自己有 `DeletedAt *time.Time` 字段，`Delete` 不会发 DELETE，而是把 `deleted_at` 更新成当前时间。

```go
db.Delete(&member)
db.Where("years = ?", 20).Delete(&Member{})
db.Where("years = ?", 20).Find(&members)          // 自动加上 deleted_at IS NULL
db.Unscoped().Where("years = ?", 20).Find(&members) // 连已软删的一起查出
```

普通查询看不到软删行。这是预期行为，不是条件写丢了。

### 物理删除

已经软删的行要真正删掉，链式调用 `Unscoped` 再 `Delete`。

```go
db.Unscoped().Delete(&bill)
```

没有 `DeletedAt` 字段的模型，普通 `Delete` 本身就是物理删除，不必再加 `Unscoped`。
