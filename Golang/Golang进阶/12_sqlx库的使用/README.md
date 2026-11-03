# sqlx：在 database/sql 之上多走一步

项目里经常用 `database/sql` 连 MySQL。`sqlx` 可以看成它的超集：连接池、预处理和事务仍由标准库负责，上面多了把行直接扫进结构体的方法，以及按名字绑定、按切片展开 `IN` 的帮手。除了常见的 `Get` 和 `Select`，本节重点看容易漏掉的 `sqlx.In` 和 `NamedExec`。

## 安装

```bash
go get github.com/jmoiron/sqlx
```

驱动仍要单独引入，例如 `github.com/go-sql-driver/mysql`。`sqlx` 不自带具体数据库的协议实现。

## 基本用法

### 连接

```go
var db *sqlx.DB

type member struct {
	ID   int    `db:"id"`
	Name string `db:"name"`
	Age  int    `db:"age"`
}

func openDB() error {
	dsn := "app:app@tcp(127.0.0.1:3306)/lab_sql?charset=utf8mb4&parseTime=true"
	var err error
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		return err
	}
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return nil
}
```

`Connect` 在 `Open` 之后立刻 `Ping`。不想在失败时返回错误、而希望直接 panic，可以用 `MustConnect`。结构体字段若和列名不一致，用 `db` 标签指出列名；没有标签时按字段名的小写形式去对。

### 查询

单行用 `Get`，它要求恰好能扫出一行，没有行时返回 `sql.ErrNoRows`。

```go
func fetchOne() {
	const q = "SELECT id, name, age FROM member WHERE id = ?"
	var m member
	if err := db.Get(&m, q, 1); err != nil {
		fmt.Printf("get: %v\n", err)
		return
	}
	fmt.Printf("id=%d name=%s age=%d\n", m.ID, m.Name, m.Age)
}
```

多行用 `Select`，目标是切片的指针。没有命中时切片保持为空，不会返回 `ErrNoRows`。

```go
func fetchMany() {
	const q = "SELECT id, name, age FROM member WHERE id > ?"
	var list []member
	if err := db.Select(&list, q, 0); err != nil {
		fmt.Printf("select: %v\n", err)
		return
	}
	fmt.Printf("list=%#v\n", list)
}
```

两个方法都必须传入指针，否则无法写回结果。

### 插入、更新、删除

`Exec` 的用法和标准库一致，返回值上仍能取自增主键和影响行数。

```go
func insertDemo() {
	const q = "INSERT INTO member(name, age) VALUES (?, ?)"
	res, err := db.Exec(q, "何安", 21)
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

func updateDemo() {
	const q = "UPDATE member SET age = ? WHERE id = ?"
	res, err := db.Exec(q, 33, 4)
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

func deleteDemo() {
	const q = "DELETE FROM member WHERE id = ?"
	res, err := db.Exec(q, 4)
	if err != nil {
		fmt.Printf("delete: %v\n", err)
		return
	}
	n, _ := res.RowsAffected()
	fmt.Printf("deleted %d\n", n)
}
```

### NamedExec

列很多时，问号的顺序容易和参数错位。`NamedExec` 用名字对应结构体字段或 map 的键。

```go
func insertNamed() error {
	const q = "INSERT INTO member (name, age) VALUES (:name, :age)"
	_, err := db.NamedExec(q, map[string]any{
		"name": "沈乔",
		"age":  27,
	})
	return err
}
```

名字前的冒号是占位标记。map 的键、或结构体的 `db` 标签，要和冒号后面的单词一致。

### NamedQuery

查询同样可以按名字绑定。结果行用 `StructScan` 扫进结构体。

```go
func queryNamed() {
	const q = "SELECT id, name, age FROM member WHERE name = :name"
	rows, err := db.NamedQuery(q, map[string]any{"name": "沈乔"})
	if err != nil {
		fmt.Printf("named: %v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var m member
		if err := rows.StructScan(&m); err != nil {
			fmt.Printf("scan: %v\n", err)
			continue
		}
		fmt.Printf("by map %#v\n", m)
	}

	probe := member{Name: "沈乔"}
	rows, err = db.NamedQuery(q, probe)
	if err != nil {
		fmt.Printf("named struct: %v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var m member
		if err := rows.StructScan(&m); err != nil {
			continue
		}
		fmt.Printf("by struct %#v\n", m)
	}
}
```

用结构体当参数时，参与绑定的是 `db` 标签而不是 json 标签。

### 事务

`Beginx` 返回 `*sqlx.Tx`。下面用 `defer` 统一处理三种结局：发生 panic 先回滚再继续抛；中途返回了错误就回滚；一切正常则提交。提交失败时把错误写回命名返回值，调用方才能看见。

```go
func transferAges() (err error) {
	tx, err := db.Beginx()
	if err != nil {
		return err
	}
	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p)
		} else if err != nil {
			tx.Rollback()
		} else {
			err = tx.Commit()
		}
	}()

	res, err := tx.Exec("UPDATE member SET age = 20 WHERE id = ?", 1)
	if err != nil {
		return err
	}
	n, err := res.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("first update touched unexpected rows")
	}

	res, err = tx.Exec("UPDATE member SET age = 50 WHERE id = ?", 5)
	if err != nil {
		return err
	}
	n, err = res.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("second update touched unexpected rows")
	}
	return nil
}
```

两条语句必须走同一个 `tx`。误用包级的 `db.Exec` 会跑到事务外面，回滚也撤不掉。

## sqlx.In

`In` 负责把切片展开成多个占位符，避免自己数问号。

### 表

```sql
CREATE TABLE member (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(20) DEFAULT '',
    age INT DEFAULT 0,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 结构体

```go
type Member struct {
	Name string `db:"name"`
	Age  int    `db:"age"`
}
```

### 占位符不是拼接

问号这一类标记叫 bindvar，只负责把值送给服务器，从而避开注入。标准库不会检查 SQL 文本是否合法，它把语句和编码后的参数原样送出。不同数据库的写法不同：

- MySQL 用 `?`
- PostgreSQL 用 `$1`、`$2`
- SQLite 两者都接受
- Oracle 用 `:name`

占位符不能改变语句结构，所以下面两句不会按想象工作：表名和列名被当成字符串值，而不是标识符。

```go
db.Query("SELECT * FROM ?", "member")
db.Query("SELECT ?, ? FROM member", "name", "city")
```

表名若必须动态选择，只能在白名单里挑一个常量拼进语句，不能把用户输入直接放进去。

### 自己拼批量插入

有几行就拼几组 `(?, ?)`，值和占位符分开存放。

```go
func insertBatch(rows []*Member) error {
	holders := make([]string, 0, len(rows))
	args := make([]any, 0, len(rows)*2)
	for _, row := range rows {
		holders = append(holders, "(?, ?)")
		args = append(args, row.Name, row.Age)
	}
	q := fmt.Sprintf("INSERT INTO member (name, age) VALUES %s", strings.Join(holders, ","))
	_, err := db.Exec(q, args...)
	return err
}
```

行数为 0 时不要执行，否则 SQL 会以 `VALUES` 结尾。

### 让 In 展开

前提是类型实现 `driver.Valuer`，`In` 会调用 `Value` 把一个元素展开成多个参数。

```go
func (m Member) Value() (driver.Value, error) {
	return []any{m.Name, m.Age}, nil
}

func insertBatchIn(rows []any) error {
	q, args, err := sqlx.In(
		"INSERT INTO member (name, age) VALUES (?), (?), (?)",
		rows...,
	)
	if err != nil {
		return err
	}
	fmt.Println(q)
	fmt.Println(args)
	_, err = db.Exec(q, args...)
	return err
}
```

问号组数要和 `rows` 的长度一致。长度不固定时，更适合用上一节自己拼，或者用下面的 `NamedExec`。

### 用 NamedExec 一次插入多行

较新的 sqlx 允许把结构体切片交给 `NamedExec`，每行对应一组命名参数。若手头的版本还没有这个行为，需要使用包含该改动的提交，而不是随便锁一个旧标签。

```bash
go get github.com/jmoiron/sqlx@master
```

```go
func insertBatchNamed(rows []*Member) error {
	_, err := db.NamedExec(
		"INSERT INTO member (name, age) VALUES (:name, :age)",
		rows,
	)
	return err
}
```

三种写法可以放在同一个 `main` 里对照。注意第二种要传值而不是指针，因为 `Value` 定义在值接收者上；传指针时方法集里同样有 `Value`，两种都能匹配接口。

```go
func main() {
	if err := openDB(); err != nil {
		panic(err)
	}
	defer db.Close()
	a := Member{Name: "沈乔", Age: 18}
	b := Member{Name: "何安", Age: 28}
	c := Member{Name: "梁舟", Age: 36}

	if err := insertBatch([]*Member{&a, &b, &c}); err != nil {
		fmt.Println("manual", err)
	}
	if err := insertBatchIn([]any{a, b, c}); err != nil {
		fmt.Println("in", err)
	}
	if err := insertBatchNamed([]*Member{&a, &b, &c}); err != nil {
		fmt.Println("named", err)
	}
}
```

### 用 In 做查询

目标语句形如 `SELECT ... WHERE id IN (3, 2, 1)`。切片长度在运行时才知道，不能预先写死问号个数。

```go
func fetchByIDs(ids []int) (list []Member, err error) {
	q, args, err := sqlx.In("SELECT name, age FROM member WHERE id IN (?)", ids)
	if err != nil {
		return nil, err
	}
	q = db.Rebind(q)
	err = db.Select(&list, q, args...)
	return list, err
}
```

`In` 返回的仍是问号风格。换到 PostgreSQL 时要 `Rebind`，把问号改成 `$1` 这种序号。MySQL 上 `Rebind` 基本是原样返回，写成习惯没有坏处。空切片会让 `In` 返回错误，调用前要先判断。

### 保持给定的顺序

`IN` 不保证结果按传入顺序排列。MySQL 可以用 `FIND_IN_SET` 把顺序固定成给定的标识序列：

```go
func fetchOrdered(ids []int) (list []Member, err error) {
	text := make([]string, 0, len(ids))
	for _, id := range ids {
		text = append(text, strconv.Itoa(id))
	}
	q, args, err := sqlx.In(
		"SELECT name, age FROM member WHERE id IN (?) ORDER BY FIND_IN_SET(id, ?)",
		ids, strings.Join(text, ","),
	)
	if err != nil {
		return nil, err
	}
	q = db.Rebind(q)
	err = db.Select(&list, q, args...)
	return list, err
}
```

标识序列是拼进 SQL 的文本，只能由程序自己的整数转换产生，不能直接采用用户原文。另一个做法是先用 `IN` 查出来，再在内存里按原始 `ids` 排序，这样不依赖某个数据库的函数。

## 对照时记住

- `Get` 没有行是错误，`Select` 没有行是空切片。
- 命名参数和问号参数不要混在同一条自己拼接的语句里，除非清楚驱动会如何处理。
- 批量插入包在一个事务里，中途失败才能整批撤回。
