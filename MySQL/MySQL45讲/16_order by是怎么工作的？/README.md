业务列表几乎都要按某个字段排序后再分页。这里用居民表做例子：查出区县为“成都”的人，按姓名排序，返回前 600 人的区县、姓名和年龄。表结构摘录如下：

```sql
CREATE TABLE resident (
  rid INT NOT NULL,
  district VARCHAR(20) NOT NULL,
  person_name VARCHAR(24) NOT NULL,
  years INT NOT NULL,
  street VARCHAR(160) DEFAULT NULL,
  PRIMARY KEY (rid),
  KEY idx_district (district)
) ENGINE=InnoDB;
```

语句是：

```sql
SELECT district, person_name, years
FROM resident
WHERE district = '成都'
ORDER BY person_name
LIMIT 600;
```

条件列上有索引，看起来不会全表扫。真正的执行还要处理排序，而且排序用的内存参数会改变算法。

# 全字段排序

`district` 上有 `idx_district` 之后，`EXPLAIN` 的 `Extra` 里常能看到 `Using filesort`。这表示服务器要排序。每个线程会分到一块叫 `sort_buffer` 的内存来放待排序的行。

`idx_district` 上，区县等于“成都”的那些索引项是连在一起的一段，每项带着对应的主键 `rid`。这段的两端可以记成从某个 `rid` 开始，到区县不再是成都的前一项结束。假设这段一共 3500 行。

全字段排序的过程是：

1. 准备 `sort_buffer`，确定要放入 `district`、`person_name`、`years`。
2. 在 `idx_district` 上找到第一个“成都”，取出主键。
3. 用主键回表，把这三列放进 `sort_buffer`。
4. 沿着区县索引取下一个主键。
5. 重复 3 和 4，直到区县不再满足条件。
6. 对 `sort_buffer` 里的行按 `person_name` 做快速排序。
7. 按排好的顺序取前 600 行返回。

排序可能完全发生在内存里，也可能要借助磁盘上的临时文件，分界是待排序数据的体积和 `sort_buffer_size`。这个参数就是排序内存的上限。数据放得下，就在内存里排完。放不下，就拆成多段，每段排好后写到临时文件，再做归并。

可以用优化器跟踪确认有没有临时文件。下面这组语句只影响当前会话：

```sql
SET optimizer_trace = 'enabled=on';

SELECT VARIABLE_VALUE INTO @rows_before
FROM performance_schema.session_status
WHERE VARIABLE_NAME = 'Innodb_rows_read';

SELECT district, person_name, years
FROM resident
WHERE district = '成都'
ORDER BY person_name
LIMIT 600;

SELECT * FROM information_schema.OPTIMIZER_TRACE\G

SELECT VARIABLE_VALUE INTO @rows_after
FROM performance_schema.session_status
WHERE VARIABLE_NAME = 'Innodb_rows_read';

SELECT @rows_after - @rows_before AS innodb_rows_delta;
```

在跟踪结果里看 `number_of_tmp_files`。它表示归并时使用的临时文件个数。假如这里是 8，可以理解为：内存一次摆不下这 3500 行，服务器把数据分成 8 段，每段单独排好并落到临时文件，再把 8 个有序文件合成一个有序结果。

`sort_buffer_size` 大于待排序数据时，这个值是 0，排序在内存里完成。参数越小，切分的段越多，临时文件个数越大。

同一段跟踪里还有两个有用的字段。`examined_rows` 为 3500，表示参与排序的就是这 3500 行，而不是最后返回的 600 行。`sort_mode` 里的 `packed_additional_fields` 表示字符串按实际长度打包。`person_name` 虽然定义成 `varchar(24)`，排序内存按内容占用，而不是每人固定留 24 个字符。

最后那个差值在这个例子里是 3500，说明执行过程中 InnoDB 只读了参与排序的那些行。为了不让跟踪语句本身干扰计数，实验里把 `internal_tmp_disk_storage_engine` 设成 MyISAM。若保持默认的 InnoDB，查询 `OPTIMIZER_TRACE` 时用到的临时表会再让 `Innodb_rows_read` 加 1，差值会变成 3501。

# rowid 排序

全字段排序只把原表读了一遍，后面的工作都在排序内存和临时文件里。它的弱点是：查询要带回的列越多，每一行在 `sort_buffer` 里越宽，内存里同时放得下的行越少，临时文件就越多，排序变差。单行很大时，这个算法不划算。

服务器用 `max_length_for_sort_data` 决定何时放弃全字段。单行超过这个长度，就换成另一种算法。这三个返回列的定义长度加起来是 20 + 24 + 4 = 48。把参数调到 20：

```sql
SET max_length_for_sort_data = 20;
```

新算法放进 `sort_buffer` 的只有排序列 `person_name` 和主键 `rid`。排完之后手里没有区县和年龄，不能直接返回，流程变成：

1. 准备 `sort_buffer`，只放 `person_name` 和 `rid`。
2. 在区县索引上找到第一个“成都”，取出主键。
3. 回表，把姓名和主键放进排序内存。
4. 取下一个主键。
5. 重复，直到区县条件结束。
6. 按姓名排序。
7. 按排序结果取前 600 个主键，再逐个回表，取出区县、姓名和年龄，返回给客户端。

和第 1 种流程相比，多出来的是第 7 步那 600 次回表。结果集在这里是逻辑说法：服务器从排好的缓冲里取出主键，回表拿到三列，直接发给客户端，不必再把 600 行完整结果在服务端攒成一张大表。

此时 `examined_rows` 仍是 3500，参与排序的行数没变。`Innodb_rows_read` 的差值则变成 4100：排序阶段读了 3500 行，排序结束后又为了 `LIMIT 600` 回表读了 600 行。

跟踪里另外两处也会变。`sort_mode` 变成 `<sort_key, rowid>`，表示排序缓冲里是姓名加主键。`number_of_tmp_files` 可能从 8 降到 6，因为行数还是 3500，每一行变窄了，总字节数下降，需要的临时文件更少。

# 全字段排序和 rowid 排序怎么取舍

两种算法可以收成一句话：内存紧张、单行又宽时，才退到 rowid 排序，换来排序阶段能多放一些行，代价是排完再回表。内存足够时，优先把要返回的列放进 `sort_buffer`，排完直接输出，少一次回表。

这也是服务器侧一个稳定的偏向：内存够用，就用内存去换磁盘访问。对 InnoDB 来说，rowid 排序多出来的回表常常就是额外的页读取，所以不会被当成第一选择。

排序本身不便宜。并不是每条 `ORDER BY` 都必须排。上面要排序，是因为从 `idx_district` 取出来的行只保证区县相同，姓名仍然是乱的。

如果从索引里取“下一条”时，姓名天然递增，就可以不再做 filesort。给居民表加上区县和姓名的联合索引：

```sql
ALTER TABLE resident ADD INDEX idx_district_name (district, person_name);
```

在这棵索引上，仍然用树查找定位到第一个“成都”。此后顺着叶子往下走，只要区县还是成都，姓名就是有序的。流程变成：

1. 在 (`district`, `person_name`) 上找到第一个满足区县条件的主键。
2. 回表取出三列，直接作为结果的一行返回。
3. 取联合索引的下一项。
4. 重复 2 和 3，直到凑满 600 行，或者区县不再是成都。

这个过程没有临时表，也没有排序。`EXPLAIN` 的 `Extra` 里不再出现 `Using filesort`。因为索引已经按姓名排好，也不必把 3500 行都读完，找到 600 行就可以停。扫描次数从 3500 降到 600。

还能再减掉回表。覆盖索引指索引上的列已经够回答查询，不必再访问主键叶子。针对这三条返回列，可以建：

```sql
ALTER TABLE resident
  ADD INDEX idx_district_name_years (district, person_name, years);
```

同一区县下仍然按姓名递增。执行时：

1. 在 (`district`, `person_name`, `years`) 上找到第一个“成都”，直接取出这三列返回。
2. 取下一项，同样直接返回三列。
3. 重复，直到 600 行或区县条件结束。

`EXPLAIN` 的 `Extra` 会出现 `Using index`，表示用了覆盖索引，连主键查找都省了。

这不是说查询里出现过的列都要塞进联合索引。每多一条索引，插入和更新都要维护它。是否为这条列表专门建覆盖索引，要看它的调用频率和写入代价。

# 小结

`ORDER BY` 至少有三条路：把要返回的列放进排序内存做全字段排序；行太宽时只排排序键和主键，再回表；或者让索引顺序本身满足 `ORDER BY`，从而去掉 filesort。有 `LIMIT` 时，有序索引还可以提前停止，不必扫完所有满足过滤条件的行。写语句时要能估计最坏情况下会扫多少行、会不会落临时文件，而不是只看结果对不对。

思考题：表上已经有 (`district`, `person_name`) 这条联合索引。现在要查成都和西安两地的居民，按姓名排序，返回前 80 行：

```sql
SELECT rid, district, person_name, years
FROM resident
WHERE district IN ('成都', '西安')
ORDER BY person_name
LIMIT 80;
```

执行时还会排序吗？原因是什么？如果应用代码由你来写，怎样让数据库侧不再做这次排序？若还要翻到很深的一页，相当于 `LIMIT 8000, 80`，方案又要怎么改？

# 上期问题时间

上一讲问的是：更新语句写回去的值若和当前存储的值相同，MySQL 会不会还是做一次修改。三个选项都有人选，需要用实验把它们分开。

表 `ticket_seq` 当前一行是 (`tid`, `score`) = (4, 9)。

第一个选项：服务器读出数据，发现值没变，就不调用引擎，直接返回。可以用锁来否定它。会话 A 执行 `UPDATE ticket_seq SET score = 9 WHERE tid = 4`，也就是把分数写成它已经具有的 9。会话 B 再更新同一行时会被堵住。加锁是 InnoDB 才能做的事，说明执行没有在进入引擎之前结束。

第二个选项：服务器调用了引擎，但引擎发现值相同，就不生成新版本，直接返回。用可见性来检验。仍从 (4, 9) 开始。会话 A 开启一致性快照并读到 (4, 9)。会话 B 把分数改成 15 并提交。会话 A 再执行 `UPDATE ticket_seq SET score = 15 WHERE tid = 4`。这条语句要写的 15，和 B 已经提交的当前值相同。若引擎因此什么都不做，A 里随后的一致性读仍应看到自己的旧快照 (4, 9)，因为它看不见 B 的提交。实际却读到 (4, 15)。这个新版本只能是 A 自己的更新生成的：快照读看不见别人的提交，但看得见本事务写下的版本。所以引擎并没有因为“数值碰巧一样”就跳过更新。

因此成立的是第三个选项：InnoDB 会认真执行这次修改，该加的锁会加，该写的新版本会写。

看起来像是白做了一次。服务器其实没有足够的信息提前跳过。这条语句读条件里能确定的是 `tid = 4`，要写的是 `score = 15`。只靠这两点看不出存储中的分数已经是 15。当前读发生在引擎内部，比较也发生在那之后。

上面的结论对应 `binlog_format = statement`。若格式是 row，并且 `binlog_row_image = FULL`，binlog 需要记下整行，服务器在读行时就会把各列都取出来。按“已经读到了就可以比较”的规则，它能发现分数无需变化，于是不再生成新版本。同样的步骤里，A 的查询结果会停在 (4, 9)。`binlog_row_image = NOBLOB` 会读出除 blob 以外的列，这个例子里没有 blob，结果同样是保持 (4, 9)。这套判断是 5.6 引入的。

若表上有 `timestamp` 且打开了自动更新，修改其他列时服务器会把相关列一并读入。读入之后就能比较，发现显式赋值并没有改变原值，从而不去写那次无意义的新版本。
