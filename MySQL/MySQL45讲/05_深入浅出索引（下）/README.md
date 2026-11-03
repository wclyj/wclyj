# 覆盖索引、最左前缀和索引下推

上一篇把 InnoDB 的 B+ 树和回表讲完了。这一篇用一条范围查询把回表次数数清楚，再看三种少回表、少建索引的办法：覆盖索引、最左前缀、索引下推。

先看这张表：

```sql
create table parcel (
  parcel_no int primary key,
  route_code int not null default 0,
  tag varchar(16) not null default '',
  index (route_code)
) engine=InnoDB;

insert into parcel values
  (10, 8, '北'),
  (30, 15, '沪'),
  (50, 22, '穗'),
  (80, 41, '蓉'),
  (120, 63, '汉'),
  (150, 70, '厦');
```

问题：执行 `select * from parcel where route_code between 22 and 63`，要搜索几次树，会碰到多少条索引记录？

主键树的叶子是整行，路线码树的叶子只有 `(route_code, parcel_no)`。走法如下：

1. 在路线码树上定位 `route_code = 22`，得到 `parcel_no = 50`。
2. 回主键树取出 50 这一行。
3. 路线码树的下一条是 41，主键 80。
4. 回主键树取出 80。
5. 下一条是 63，主键 120。
6. 回主键树取出 120。
7. 再下一条是 70，已经大于 63，停止。

路线码树读了 4 条（22、41、63、70），其中 70 只是用来判断越界。真正回表 3 次。回表指的就是拿着二级索引里的主键，再去聚簇索引取整行。

这条语句要的是 `select *`，收件标签只在主键叶子上，所以绕不开回表。如果查询能被二级索引本身满足，第二次搜索就可以省掉。

## 覆盖索引

把语句改成：

```sql
select parcel_no from parcel where route_code between 22 and 63;
```

需要的主键已经躺在路线码树的叶子里。引擎沿叶子链表读完区间，直接把 `parcel_no` 交回去，不再访问主键树。这条索引覆盖了查询，叫做覆盖索引。

少搜一棵树，随机读会明显下降，所以“尽量让高频语句被索引覆盖”是常见的优化动作。

注意行数口径。引擎在路线码树上实际看过 4 条索引项，但 Server 层只收到 3 行结果。慢日志里的扫描行数按 Server 拿到的行计，这里会显示 3，而不是引擎内部读过的 4。看执行计划时要记得这两个数字不是一回事。

覆盖索引经常要多建一列，空间和维护都要付钱。例如读者表：

```sql
create table reader (
  reader_id int not null,
  card_no varchar(32) default null,
  reader_name varchar(32) default null,
  grade int default null,
  campus_id tinyint default null,
  primary key (reader_id),
  key card_no (card_no),
  key name_grade (reader_name, grade)
) engine=InnoDB;
```

证件号已经唯一标识一名读者。若业务只是凭证件号查整行，单列索引够用，再加 `(card_no, reader_name)` 看起来像浪费。

如果有一条很高频的接口，只根据证件号取姓名，联合索引就有意义：叶子里同时有证件号、姓名和主键，这条语句不再回表。是否建这个冗余索引，取决于这条接口省下的回表，能不能抵消每次写入都要维护第二棵树的成本。这是业务建模时要做的取舍，不是索引越多越好。

## 最左前缀

不能为每一种条件各建一棵树。读者偶尔会按证件号查家庭住址，这种请求很少，单独建 `(card_no, address)` 又显得奢侈，全表扫描也不甘心。

B+ 树可以靠最左前缀定位。用已有的 `(reader_name, grade)` 来看：索引项先按姓名排，姓名相同再按年级排。

- 查姓名等于「林夏」：直接定位到这个姓名的第一项，向后扫到姓名变化为止。
- 查姓「林」、也就是 `reader_name like '林%'`：同样能定位到第一个「林」开头的项，再顺序扫，直到前缀不再是「林」。

所以最左前缀既可以是联合索引左边的若干列，也可以是某一列字符串左边的若干个字符。`like '%夏'` 这种左边不确定的写法用不上这棵树，因为有序性从左边开始才成立。

已经有了 `(reader_name, grade)`，就不必再单独建 `reader_name`。调整列顺序若能少维护一棵树，这个顺序优先。前面的高频需求“按证件号取姓名”，建成 `(card_no, reader_name)` 之后，偶尔按证件号查住址也可以用这棵树定位证件号，再回表读住址，不必为低频请求另建索引。

如果既有 `a and b` 的组合条件，又有只按 `a`、只按 `b` 的查询，只建 `(a, b)` 覆盖不了单独的 `b`。只含 `b` 的条件无法使用 `(a, b)`，还得再留一棵 `b` 的索引。

这时比的是空间。姓名比年级宽。建 `(reader_name, grade)` 再加一棵窄的 `(grade)`，比建 `(grade, reader_name)` 再加一棵宽的 `(reader_name)` 更省：多出来的那棵单列索引应落在更短的列上。联合索引本身哪边在左，还是由“能不能少建一棵”决定，空间原则用来处理剩下的那棵单列索引。

## 索引下推

最左前缀用完之后，索引里其余的列怎么办？

需求是：姓「林」、年级为 6、并且校区编号为 2 的读者。

```sql
select *
from reader
where reader_name like '林%'
  and grade = 6
  and campus_id = 2;
```

`林%` 能在 `(reader_name, grade)` 上定位起点，但年级并不是一个确定的等值前缀，校区编号根本不在这棵索引里。5.6 之前，引擎从第一个「林」开始，每条都回表，取出整行再判断年级和校区。假设这个前缀下有 5 行，就要回表 5 次，其中年级不是 6 的也会回表。

5.6 引入了索引下推（index condition pushdown）。遍历二级索引时，引擎可以先用索引里已经有的列做判断。年级就在 `(reader_name, grade)` 的叶子上，不等于 6 的直接跳过。假设 5 行里只有 2 行年级是 6，那就只对这 2 行回表，回表之后再检查 `campus_id`。回表次数从 5 降到 2。

下推过滤不了索引里没有的列。`campus_id` 仍然要等回到主键树才能看。能下推的，是那些已经包含在当前索引条目中的条件。

## 小结

回表是二级索引查整行的额外树搜索。覆盖索引让结果列都出现在二级索引里，把回表消掉。最左前缀让一棵联合索引服务多种左边对齐的条件，从而少建索引。索引下推则在不得不回表之前，先用索引内的列把行丢掉。

设计表时的目标可以收成一句：在满足语句的前提下，少访问页、少维护树。

## 联合主键已经包含 a、b，为什么还要 ca 和 cb

接手一张历史表，结构类似：

```sql
create table slot (
  bay int not null,
  rack int not null,
  lane int not null,
  flag int not null,
  primary key (bay, rack),
  key lane (lane),
  key lane_bay (lane, bay),
  key lane_rack (lane, rack)
) engine=InnoDB;
```

业务坚持主键必须是 `(bay, rack)`。同事说下面两句所以要后两个索引：

```sql
select * from slot where lane = 9 order by bay limit 1;
select * from slot where lane = 9 order by rack limit 1;
```

这个解释只对一半。

InnoDB 的二级索引叶子会带上主键的全部列，并且在索引列相同的时候按主键列继续排序。因此单列索引 `lane` 的实际顺序是 `(lane, bay, rack)`，不是只有 `lane`。

`where lane = 9 order by bay limit 1` 要的顺序，单列 `lane` 已经给出：同一个 `lane` 下面先按 `bay` 再按 `rack`。`lane_bay` 的顺序也是 `(lane, bay, rack)`，和 `lane` 重复，可以不建。

`where lane = 9 order by rack limit 1` 要在相同 `lane` 下按 `rack` 取第一条。`lane` 和 `lane_bay` 都是先排 `bay`，不能满足这个顺序，除非把同一个 `lane` 的行都取出来再排序。`lane_rack` 的顺序是 `(lane, rack, bay)`（`bay` 作为主键里还没出现的列被补在最后），正好能按 `rack` 停下。所以 `lane_rack` 要留，`lane_bay` 不必留。

## 上一讲的重建索引

二级索引因为删除和页分裂会变疏。丢掉再按顺序建一棵新树，页利用率最高，那对 `drop index` 加 `add index` 是合理的。

主键不是一棵可以单独拆掉的附属结构。删除主键、再加回主键，每一步都在重建整张表。两句连着跑，前一句的工作被后一句覆盖。改成一句 `alter table ... engine=InnoDB`，表只重写一次，空洞同样会被收掉。
