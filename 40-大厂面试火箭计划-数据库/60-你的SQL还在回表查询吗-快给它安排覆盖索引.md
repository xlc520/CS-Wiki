# 你的 SQL 还在回表查询吗？快给它安排覆盖索引

### 什么是回表查询

小伙伴们可以先看这篇文章了解下什么是聚集索引和辅助索引：[Are You OK？主键、聚集索引、辅助索引](https://mp.weixin.qq.com/s/ofGWt_kAajIB0EpI67pbZg)，简单回顾下，聚集索引的叶子节点包含完整的行数据，而非聚集索引的叶子节点存储的是每行数据的辅助索引键 + 该行数据对应的聚集索引键（主键值）。

假设有张 user 表，包含 id（主键），name，age（普通索引）三列，有如下数据：

```sql
id	name	    age
1	Jack	    18
7	Alice	    28
10	Bob	    	38
20	Carry	    48
```

画一个比较简单比较容易懂的图来看下聚集索引和辅助索引：

- 聚集索引：

  ![](https://gitee.com/veal98/images/raw/master/img/20210830100105.png)

- 辅助索引（age）：

  ![](https://gitee.com/veal98/images/raw/master/img/20210830095343.png)

如果查询条件为主键，则只需扫描一次聚集索引的 B+ 树即可定位到要查找的行记录。举个例子：

```sql
select * from user where id = 7;
```

查找过程如图中绿色所示：

![](https://gitee.com/veal98/images/raw/master/img/20210830095538.png)

如果查询条件为普通索引（辅助索引） age，则需要先查一遍辅助索引 B+ 树，根据辅助索引键得到对应的聚集索引键，然后再去聚集索引 B+ 树中查找到对应的行记录。举个例子：

```sql
select * from user where age = 28;
```

上述 `select *` 等同于 `select id, age, name` 对吧，id 是主键索引，age 是普通索引，而 name 并不存在于 age 索引的 B+ 树上，所以通过 age 索引查询到 id 和 age 的值之后，还需要去聚集索引上才能查到 name 的值。

如图所示，第一步，查 age 辅助索引：

![](https://gitee.com/veal98/images/raw/master/img/20210830095824.png)

第二步，查聚集索引：

![](https://gitee.com/veal98/images/raw/master/img/20210830100105.png)

这就是所谓的**回表查询**，因为需要**扫描两次索引 B+ 树**，所以很显然它的性能较扫一遍索引树更低。

### 什么是覆盖索引

覆盖索引的目的就是避免发生回表查询，也就是说，通过覆盖索引，只需要扫描一次 B+ 树即可获得所需的行记录。

### 如何实现覆盖索引

上文解释过，下面这个 SQL 语句需要查询两次 B+ 树：

```sql
select * from user where age = 28;
```

我们将其稍作修改，使其只需要查询一次 B+ 树：

```sql
select id from user where age = 28;
```

之前我们的返回结果是整个行记录，现在我们的返回结果只需要 id

id 是什么？主键索引（聚集索引），age 是什么？普通索引（辅助索引），age 索引的 B+ 树的叶子节点存储的是什么？辅助索引键 + 对应的聚集索引键

所以这条 SQL 语句只需要扫描一次 age 索引的 B+ 树就可以直接提供查询结果，不需要回表。也就是说，在这个查询里面，索引 age 已经 **覆盖了** 我们的查询需求，我们称为覆盖索引（所以覆盖索引其实就是一种联合索引）。

![](https://gitee.com/veal98/images/raw/master/img/20210830095824.png)

由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

再回到这条语句：

```sql
select * from user where age = 28;
```

如果想要使用覆盖索引对这条语句进行优化，该如何做？

很简单了，对吧，我们只需要把 `age,name` 设置为联合索引就可以了：

```sql
create index idx_age_name on user(`age`,`name`);
```

此时 age 和 name 作为辅助索引键都在同一棵辅助索引的 B+ 树上，所以只需扫描一次这个组合索引的 B+ 树即可获取到 id、age 和 name，这就是实现了索引覆盖。

### 覆盖索引的常见使用场景

在下面三个场景中，可以使用覆盖索引来进行优化 SQL 语句：

1）**列查询回表优化**（如上面讲的例子，将单列索引 age 升级为联合索引（age, name））

2）**全表 count 查询**

举个例子，假设 user 表中现在只有一个索引即主键 id：

```sql
select count(age) from user;
```

可以用 explain 分析下这条语句，如果 Extra 字段为 Using index 时，就表示触发索引覆盖：

![](https://gitee.com/veal98/images/raw/master/img/20210902095054.png)

显然现在是没有触发覆盖索引的，我们来优化下：将 age 列设置为索引 `create index idx_age on user(age)`，这样只需要查一遍 age 索引的 B+ 树即可得到结果：

![](https://gitee.com/veal98/images/raw/master/img/20210902095542.png)

3）**分页查询**

```sql
select id, age, name from user order by username limit 500, 100;
```

对于这条 SQL，因为 name 字段不是索引，所以在分页查询需要进行回表查询。

![](https://gitee.com/veal98/images/raw/master/img/20210902095728.png)

**Using filesort** 表示没有使用索引的排序，或者说表示在索引之外，需要额外进行外部的排序动作。看到这个字段就应该意识到你需要对这条 SQL 进行优化了。

使用索引覆盖优化：将 (age, name) 设置为联合索引，这样只需要查一遍 (age, name) 联合索引的 B+ 树即可得到结果。

![](https://gitee.com/veal98/images/raw/master/img/20210902100000.png)

> 我是小牛肉，长风破浪会有时，小伙伴们下篇文章再见 👋

