MySQL闪回原理与实战
========================
mysqlbinlog  /var/lib/mysql/mysql-bin.000007 --database=test --start-datetime='2017-12-29 10:00:01' --stop-datetime='2017-12-29 11:25:59'   > 29-day.sql


DBA或开发人员，有时会误删或者误更新数据，如果是线上环境并且影响较大，就需要能快速回滚。传统恢复方法是利用备份重搭实例，再应用去除错误sql后的binlog来恢复数据。此法费时费力，甚至需要停机维护，并不适合快速回滚。也有团队利用LVM快照来缩短恢复时间，但快照的缺点是会影响mysql的性能。

MySQL闪回(flashback)利用binlog直接进行回滚，能快速恢复且不用停机。

闪回原理
===

**binlog概述**

MySQL binlog以event的形式，记录了MySQL server从启用binlog以来所有的变更信息，能够帮助重现这之间的所有变化。MySQL引入binlog主要有两个目的：一是为了主从复制；二是某些备份还原操作后需要重新应用binlog。

有三种可选的binlog格式，各有优缺点：

* statement：基于SQL语句的模式，binlog数据量小，但是某些语句和函数在复制过程可能导致数据不一致甚至出错；
* row：基于行的模式，记录的是行的完整变化。很安全，但是binlog会比其他两种模式大很多；
* mixed：混合模式，根据语句来选用是statement还是row模式；

利用binlog闪回，**需要将binlog格式设置为row**。row模式下，一条使用innodb的insert会产生如下格式的binlog：

```
# at 1129
#161225 23:15:38 server id 3773306082  end_log_pos 1197         Query   thread_id=1903021       exec_time=0     error_code=0
SET TIMESTAMP=1482678938/*!*/;
BEGIN
/*!*/;
# at 1197
#161225 23:15:38 server id 3773306082  end_log_pos 1245         Table_map: `test`.`user` mapped to number 290
# at 1245
#161225 23:15:38 server id 3773306082  end_log_pos 1352         Write_rows: table id 290 flags: STMT_END_F

BINLOG '
muJfWBPiFOjgMAAAAN0EAAAAACIBAAAAAAEABHRlc3QABHVzZXIAAwMPEQMeAAAC
muJfWB7iFOjgawAAAEgFAAAAACIBAAAAAAEAAgAD//gBAAAABuWwj+i1tVhK1hH4AgAAAAblsI/p
krFYStYg+AMAAAAG5bCP5a2ZWE/onPgEAAAABuWwj+adjlhNeAD4BQAAAAJ0dFhRYJM=
'/*!*/;
# at 1352
#161225 23:15:38 server id 3773306082  end_log_pos 1379         Xid = 5327954
COMMIT/*!*/;
```

**闪回原理**

> 既然binlog以event形式记录了所有的变更信息，那么我们把需要回滚的event，从后往前回滚回去即可。

对于单个event的回滚，我们以表test.user来演示原理

```
mysql> show create table test.user\G
*************************** 1. row ***************************
       Table: user
Create Table: CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8
```
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2013-11-11 00:00:00', 4, 'xiaoli'); 
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2016-11-11 20:25:00', 3, 'xiaosun'); 
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2014-11-11 00:04:48', 2, 'xiaoqian'); 
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2013-11-11 00:04:33', 1, 'xiaozhao');
	
	
	
* 对于delete操作，我们从binlog提取出delete信息，生成的回滚语句是insert。(注：为了方便解释，我们用binlog2sql将原始binlog转化成了可读SQL)
	
	```
	原始：DELETE FROM `test`.`user` WHERE `id`=1 AND `name`='小赵';
	回滚：INSERT INTO `test`.`user`(`id`, `name`) VALUES (1, '小赵');
	```

* 对于insert操作，回滚SQL是delete。
	
	```
	原始：INSERT INTO `test`.`user`(`id`, `name`) VALUES (2, '小钱');
	回滚：DELETE FROM `test`.`user` WHERE `id`=2 AND `name`='小钱';
	```

* 对于update操作，回滚sql应该交换SET和WHERE的值。
	
	```
	原始：UPDATE `test`.`user` SET `id`=3, `name`='小李' WHERE `id`=3 AND `name`='小孙';
	回滚：UPDATE `test`.`user` SET `id`=3, `name`='小孙' WHERE `id`=3 AND `name`='小李';
	```



闪回实战
===

> 真实的闪回场景中，最关键的是能快速筛选出真正需要回滚的SQL。
首先安装：
```
shell> git clone https://github.com/danfengcao/binlog2sql.git && cd binlog2sql
shell> pip install -r requirements.txt
```

离线环境则需要用pip install *.gz分别安装，三个包已经包含在主文件夹里面

**背景**：小明在11:44时误删了test库user表部分数据，需要紧急回滚。

```bash
test库user表原有数据
mysql> select * from user;
+----+----------+---------------------+
| id | name     | addtime             |
+----+----------+---------------------+
|  1 | xiaozhao | 2013-11-11 00:04:33 |
|  2 | xiaoqian | 2014-11-11 00:04:48 |
|  3 | xiaosun  | 2016-11-11 20:25:00 |
|  4 | xiaoli   | 2013-11-11 00:00:00 |
+----+----------+---------------------+
4 rows in set (0.00 sec)


11:44时，user表大批数据被误删除。与此同时，正常业务数据是在继续写入的
delete from user where addtime>'2014-01-01';
Query OK, 2 rows affected (0.02 sec)

mysql> select count(*) from user;
+----------+
| count(*) |
+----------+
|        2 |
+----------+
1 row in set (0.00 sec)

mysql> select * from user;
+----+----------+---------------------+
| id | name     | addtime             |
+----+----------+---------------------+
|  1 | xiaozhao | 2013-11-11 00:04:33 |
|  4 | xiaoli   | 2013-11-11 00:00:00 |
+----+----------+---------------------+
2 rows in set (0.00 sec)
```

**恢复数据步骤**：

1. 登录mysql，查看目前的binlog文件

	```bash
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       475 |
| mysql-bin.000002 | 449038846 |
| mysql-bin.000003 | 568962489 |
| mysql-bin.000004 |       177 |
| mysql-bin.000005 |      7859 |
| mysql-bin.000006 |       737 |
| mysql-bin.000007 |      7469 |
| mysql-bin.000008 |       664 |
| mysql-bin.000009 |       446 |
+------------------+-----------+
9 rows in set (0.00 sec)
	```

2. 最新的binlog文件是mysql-bin.000054。我们的目标是筛选出需要回滚的SQL，由于误操作人只知道大致的误操作时间，我们首先根据时间做一次过滤。只需要解析test库user表。(注：如果有多个sql误操作，则生成的binlog可能分布在多个文件，需解析多个文件)

	```bash
	[root@localhost binlog2sql]# ./binlog2sql.py -h127.0.0.1 -P3306 -uroot -p'Aa123456' -dtest -tuser --start-file='mysql-bin.000009' > ./raw.sql
	raw.sql输出：
	DELETE FROM `test`.`user` WHERE `addtime`='2014-11-11 00:04:48' AND `id`=2 AND `name`='xiaoqian' LIMIT 1; #start 4 end 415 time 2018-01-03 09:40:57
	DELETE FROM `test`.`user` WHERE `addtime`='2016-11-11 20:25:00' AND `id`=3 AND `name`='xiaosun' LIMIT 1; #start 4 end 415 time 2018-01-03 09:40:57
	...
	```
	参数检查./binlog2sql.py -h127.0.0.1 -P3306 -uroot -p'Aa123456' --start-file='mysql-bin.000008' > ./raw.sql
	[root@localhost binlog2sql]# ./binlog2sql.py -uroot -p'Aa123456' --start-file='mysql-bin.000008' > ./raw.sql
	usage: binlog2sql.py [-h HOST] [-u USER] [-p PASSWORD] [-P PORT]
                     [--start-file START_FILE] [--start-position START_POS]
                     [--stop-file END_FILE] [--stop-position END_POS]
                     [--start-datetime START_TIME] [--stop-datetime STOP_TIME]
                     [--stop-never] [--help] [-d [DATABASES [DATABASES ...]]]
                     [-t [TABLES [TABLES ...]]] [--only-dml]
                     [--sql-type [SQL_TYPE [SQL_TYPE ...]]] [-K] [-B]
                     [--back-interval BACK_INTERVAL]


3. 根据位置信息，我们确定了误操作sql来自同一个事务，准确位置在257427-504272之间(binlog2sql对于同一个事务会输出同样的start position)。再根据位置过滤，使用 _**-B**_ 选项生成回滚sql，检查回滚sql是否正确。(注：真实场景下，生成的回滚SQL经常会需要进一步筛选。结合grep、编辑器等)

	```bash
	[root@localhost binlog2sql]#  ./binlog2sql.py -h127.0.0.1 -P3306 -uroot -p'Aa123456' -dtest -tuser --start-file='mysql-bin.000009' --start-position=4 --stop-position=415 -B > ./rollback.sql
	rollback.sql 输出：
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2016-11-11 20:25:00', 3, 'xiaosun'); #start 4 end 415 time 2018-01-03 09:40:57
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2014-11-11 00:04:48', 2, 'xiaoqian'); #start 4 end 415 time 2018-01-03 09:40:57


	[root@localhost binlog2sql]# wc -l /tmp/rollback.sql
	2 /tmp/rollback.sql

	```
        
4. 与业务方确认回滚sql没问题，执行回滚语句。登录mysql，确认回滚成功。

	```bash
	[root@localhost binlog2sql]# mysql -P3306 -uroot -p'Aa123456' < ./rollback.sql
	mysql: [Warning] Using a password on the command line interface can be insecure.

	mysql> select * from user;
	+----+----------+---------------------+
	| id | name     | addtime             |
	+----+----------+---------------------+
	|  1 | xiaozhao | 2013-11-11 00:04:33 |
	|  2 | xiaoqian | 2014-11-11 00:04:48 |
	|  3 | xiaosun  | 2016-11-11 20:25:00 |
	|  4 | xiaoli   | 2013-11-11 00:00:00 |
	+----+----------+---------------------+
	4 rows in set (0.00 sec)

	mysql> select count(*) from user;
	+----------+
	| count(*) |
	+----------+
	|        4 |
	+----------+
	1 row in set (0.00 sec)

### TIPS

* 闪回的目标：快速筛选出真正需要回滚的数据。
* 先根据库、表、时间做一次过滤，再根据位置做更准确的过滤。
* 由于数据一直在写入，要确保回滚sql中不包含其他数据。可根据是否是同一事务、误操作行数、字段值的特征等等来帮助判断。
* 执行回滚sql时如有报错，需要查实具体原因，一般是因为对应的数据已发生变化。由于是严格的行模式，只要有唯一键(包括主键)存在，就只会报某条数据不存在的错，不必担心会更新不该操作的数据。业务如果有特殊逻辑，数据回滚可能会带来影响。
* 如果只回滚某张表，并且该表有关联表，关联表并不会被回滚，需与业务方沟通清楚。

#### **哪些数据需要回滚，让业务方来判断！**

闪回工具
===

MySQL闪回特性最早由阿里彭立勋开发，彭在2012年给官方提交了一个patch，并对[闪回设计思路](http://www.penglixun.com/tech/database/mysql_flashback_feature.html)做了说明(设计思路很有启发性，强烈推荐阅读)。但是因为种种原因，业内安装这个patch的团队至今还是少数，真正应用到线上的更是少之又少。彭之后，又有多位人员针对不同mysql版本不同语言开发了闪回工具，原理用的都是彭的思路。

我将这些闪回工具按实现方式分成了三类。

* 第一类是以patch形式集成到官方工具mysqlbinlog中。以彭提交的patch为代表。

	> 优点
	>
	> * 上手成本低。mysqlbinlog原有的选项都能直接利用，只是多加了一个闪回选项。闪回特性未来有可能被官方收录。
	> * 支持离线解析。
	>
	> 缺点
	>
	> * 兼容性差、项目活跃度不高。由于binlog格式的变动，如果闪回工具作者不及时对补丁升级，则闪回工具将无法使用。目前已有多位人员分别针对mysql5.5，5.6，5.7开发了patch，部分项目代码公开，但总体上活跃度都不高。
	> * 难以添加新功能，实战效果欠佳。在实战中，经常会遇到现有patch不满足需求的情况，比如要加个表过滤，很简单的一个需求，代码改动也不会大，但对大部分DBA来说，改mysql源码还是很困难的事。
	> * 安装稍显麻烦。需要对mysql源码打补丁再编译生成。
	
	> 这些缺点，可能都是闪回没有流行开来的原因。

* 第二类是独立工具，通过伪装成slave拉取binlog来进行处理。以binlog2sql为代表。

	> 优点
	>
	> * 兼容性好。伪装成slave拉binlog这项技术在业界应用的非常广泛，多个开发语言都有这样的活跃项目，MySQL版本的兼容性由这些项目搞定，闪回工具的兼容问题不再突出。
	> * 添加新功能的难度小。更容易被改造成DBA自己喜欢的形式。更适合实战。
	> * 安装和使用简单。
	>
	> 缺点
	>
	> * 必须开启MySQL server。

* 第三类是简单脚本。先用mysqlbinlog解析出文本格式的binlog，再根据回滚原理用正则进行匹配并替换。

	> 优点
	>
	> * 脚本写起来方便，往往能快速搞定某个特定问题。
	> * 安装和使用简单。
	> * 支持离线解析。
	>
	> 缺点
	>
	> * 通用性不好。
	> * 可靠性不好。

就目前的闪回工具而言，线上环境的闪回，笔者建议使用binlog2sql，离线解析使用mysqlbinlog。


参考资料
==============

[1] MySQL Internals Manual
, [Chapter 20 The Binary Log](http://dev.mysql.com/doc/internals/en/binary-log.html)

[2] 彭立勋，[MySQL下实现闪回的设计思路](http://www.penglixun.com/tech/database/mysql_flashback_feature.html)

[3] Lixun Peng, [Provide the flashback feature by binlog](https://bugs.mysql.com/bug.php?id=65178)

[4] 王广友，[mysqlbinlog flashback 5.6完全使用手册与原理](http://www.cnblogs.com/youge-OneSQL/p/5249736.html)

[5] 姜承尧, [拿走不谢，Flashback for MySQL 5.7](http://mp.weixin.qq.com/s?__biz=MjM5MjIxNDA4NA==&mid=2649737874&idx=1&sn=a993322ae58db541c2cf4d9a1efa3063&chksm=beb2d7b989c55eafb7ddcadb28f45bb6018b3e9e65df20b30217fe8cb26d3d444d58076f2d76&mpshare=1&scene=1&srcid=1228ta3qs3QIN6FS4AUCuCKm#rd)

[6] 林晓斌, [MySQL闪回方案讨论及实现](http://dinglin.iteye.com/blog/1539167)

[7] xiaobin lin, [flashback from binlog for MySQL](https://bugs.mysql.com/bug.php?id=65861)

[8] mariadb.com, [AliSQL and some features that have made it into MariaDB Server](https://mariadb.com/resources/blog/alisql-and-some-features-have-made-it-mariadb-server)

[9] danfengcao, [binlog2sql: Parse MySQL binlog to SQL you want](https://github.com/danfengcao/binlog2sql)
