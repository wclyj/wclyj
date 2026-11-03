# 用 explain 看清一条语句准备怎么跑

讨论索引时总会提到 `explain`。它不是再把语句执行一遍把结果算出来，而是让优化器把准备采用的计划交出来：用哪张表驱动、走什么访问类型、选了哪个索引、大概会看多少行。调优时先看计划，再决定要不要改索引或改写法。

有一个例外：如果 `from` 里带了子查询，MySQL 仍可能先把子查询跑掉，结果放进临时表，再对临时表做计划。看到 `explain` 很慢，先检查是不是这个原因，而不是以为 `explain` 本身在做全表计算。

## 怎么读一份计划

在语句前面加上 `explain`：

```sql
explain
select m.member_id, m.nick, p.email, p.city_name
from course_member m
join member_profile p on m.member_id = p.member_id
where m.cohort = 0;
```

返回的每一行对应计划里的一个查询块或一张表。下面按列说明这些字段在告诉你什么。例子里的表都换成会员和摊位，避免对着某一份旧截图对号入座。

## id

`id` 是 select 的编号，同一条语句里谁先被处理，往往体现在编号和它出现的顺序上。没有子查询时通常只有一个编号。

```sql
explain
select *
from (
  select member_id, nick
  from course_member
  limit 8
) t;
```

派生表会让内层 select 和外层 select 拿到不同的 `id`。编号相同的几行，多半是同一个 select 里的多表连接；编号更大的部分，常常是要先物化的子查询。它不决定性能，只帮你把计划分成几段。

## select_type

这一列标出每个 select 块在整句里的角色。

- `SIMPLE`：没有 `union`，也没有子查询。
- `PRIMARY`：最外层的那个 select。存在子查询时，外面那一层会标成它。
- `UNION`：`union` 里从第二段开始的 select。例如：

```sql
explain
select * from course_member limit 8
union
select * from course_member limit 8, 8;
```

第一段是 `PRIMARY`，第二段是 `UNION`，最后还可能有一行 `UNION RESULT`，表示把几段结果并起来的临时表。

- `DEPENDENT UNION`：`union` 的后段依赖外层查询的值，不能单独算完。
- `UNION RESULT`：合并 `union` 各段的那一步。

还有 `SUBQUERY`、`DERIVED`、`DEPENDENT SUBQUERY` 等取值，看到它们就知道计划里出现了子查询或派生表，值得先确认子查询是不是被反复执行。

## table

这一行在读哪张表。派生表、`union` 的结果会显示成 `<derivedN>` 或 `<unionN>` 这类名字，N 对应前面的 `id`。

## type

访问类型从好到差排，是这份计划里最值得盯的一列。下面只展开最常见、也最好的几档；再往差走是 `range`、`index`、`ALL`，分别表示索引范围扫描、整棵索引扫描和全表扫描。能停在前面几档，就不要滑到 `ALL`。

### system

表里至多一行，是 `const` 的特例，系统表上偶尔能看见，业务表里基本碰不到。

### const

通过主键或唯一索引定位，至多匹配一行。优化器把它当成常量，后面的连接可以直接使用这个值。

```sql
explain
select * from audit_event where event_id = 111;
```

`event_id` 若是主键，`type` 就是 `const`。下面这句虽然也只取一行，但没有用唯一键定位，只是扫到一行就停，类型不会是 `const`：

```sql
explain
select * from audit_event limit 1;
```

记住条件：比较的是主键或唯一索引，并且比较结果不会超过一行。

### eq_ref

连接时，对于驱动表的每一行，当前表都能用主键或唯一索引精确对上一条。手册里的说法是：索引的全部列都参与了连接，并且索引唯一。这是除 `const` 之外最好的连接方式。

```sql
explain
select *
from member_profile f
join course_member m on f.member_id = m.member_id;
```

若 `course_member.member_id` 是主键，计划里这张被驱动的表会显示 `eq_ref`：前面每来一行，这里只取出一行。

### ref

同样是等值查找，但索引不唯一，或者只用了联合索引的最左前缀，因此一个键可能对应多行。匹配行不多时，这仍然是合格的访问。它和 `eq_ref` 的分界就是：能不能保证至多一行。

## possible_keys 和 key

`possible_keys` 是优化器认为用得上的候选索引。`key` 是它最后选中的那一个。

候选里有、`key` 却是 `NULL`，说明它估完成本后放弃了索引，准备扫表。`possible_keys` 为 `NULL` 则是根本没有能用的索引。两列一起看，才能区分“没有索引”和“有索引但没被选中”。

## key_len

实际用上的索引前缀长度，单位是字节。联合索引只使用了左边几列时，`key_len` 不会等于整棵索引的宽度。用它来确认最左前缀用到了第几列，比只看 `key` 的名字可靠。字符集和是否可空都会算进字节数：可空列要额外占标志位。

## ref

这一列写的是：与 `key` 做等值比较的东西是什么。可能是 `const`，也可能是前面某张表的某一列，例如 `course_member.member_id`。它回答“索引键和谁比”，不是访问类型里的那个 `ref`。

## rows

优化器估计要检查的行数，不是结果集行数。数值越大，越说明索引没有把范围收窄。它是估算，和慢日志里的 `Rows_examined` 同方向，但不必相等。比较两种写法时，`rows` 成倍变小，通常比微差更有参考价值。

## Extra

补充动作都挤在这里。常见的几项：

- `Distinct`：对当前行组合，找到第一行匹配就不再往下找。
- `Not exists`：某种反连接的优化，外层值在内层确认不存在后就可以停。
- `Range checked for each record`：没有一棵索引能一上来就用，优化器准备对驱动表的每一行再临时看能不能用索引。这通常说明索引和条件没对上。
- `Using filesort`：排序无法靠索引顺序完成，需要额外的排序过程。结果列里若带大字段，排序行更宽，会更慢。它的含义是再做一遍排序，不是说一定在磁盘上落了文件。
- `Using index`：所需的列都在这棵索引里，不必回表读整行。这是覆盖索引，不是“用了索引”的同义词。用了索引但还要回表时，不会单靠这一项来表示。
- `Using temporary`：为了完成查询要建临时表。`group by` 和 `order by` 的列不一致时经常出现。看到它，就该检查能不能用索引顺序把排序或分组消化掉。

`Using temporary` 和连接顺序经常绑在一起。比较下面两句，条件相同，只有排序列不同：

```sql
explain
select b.booth_id
from booth b
join district d on d.booth_id = b.booth_id
where d.district_no = 310
  and b.booth_state = 'open'
order by b.booth_id desc;
```

```sql
explain
select b.booth_id
from booth b
join district d on d.booth_id = b.booth_id
where d.district_no = 310
  and b.booth_state = 'open'
order by d.booth_id desc;
```

多表连接用的是嵌套循环：先取出驱动表上满足条件的行，再拿每一行去下一张表查找，最后拼结果。`explain` 输出里排在前面的那张表就是驱动表。

若驱动表是 `district`，`order by d.booth_id` 有机会顺着驱动表已经得到的顺序往下做。`order by b.booth_id` 排序的是被驱动表的列，要等循环把结果拼完才能排，于是出现临时表和 filesort。

驱动表可以粗略理解成：写了连接条件时，套用条件后剩余行更少的那张表；完全没有连接条件时，行数更少的那张表。更稳妥的原则是让小结果集去驱动大结果集。真不确定谁小，就写成普通的等值连接，把选择权交给优化器：

```sql
select *
from booth b, district d
where b.booth_id = d.booth_id;
```

不要为了“感觉”强行写死 `straight_join`，除非 `explain` 已经证明优化器选错了。

还有一种和临时表有关的取舍。`lane_code` 上有普通索引时：

```sql
select * from cargo order by lane_code;
```

这是全表排序。优化器会想：沿索引走仍然要为每一行回表，还不如扫聚簇索引，在排序缓冲区里排。此时可能看到 filesort，并且 `key` 不是 `lane_code`。

加上范围，让它不必碰整张表：

```sql
select * from cargo where lane_code > 20 order by lane_code;
```

优化器改成沿 `lane_code` 的有序叶子走，边走边回表，顺序已经满足 `order by`，临时表和额外排序就可以消失。全表和有范围，计划可以完全不同。

## 小结

看一份 `explain`，顺序建议是：`type` 有没有掉到 `ALL`，`key` 是不是期望的那棵索引，`key_len` 用了前缀的多少，`rows` 是否过大，`Extra` 里有没有不该出现的 `Using temporary` 和 `Using filesort`。`Using index` 表示覆盖，不必回表。

连接时先确认第一行是哪张驱动表，再看 `order by` 的列属不属于它。排序列在被驱动表上，就得准备为合并后的结果付出临时表。
