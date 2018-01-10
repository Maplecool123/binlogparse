binlog2sql
========================

从MySQL binlog解析出你要的SQL。根据不同选项，你可以得到原始SQL、回滚SQL、去除主键的INSERT SQL等。
目前存在问题：
1、rollback sql 语句还没有生成哦（ok了）
2、不同数据库之间sql语句的转换，也就是不同数据库之间的接口



用途
===========

* 数据快速回滚(闪回)
* 主从切换后新master丢数据的修复
* 从binlog生成标准SQL，带来的衍生功能


项目状态
===
还在开发，请不要轻易尝试线上删除数据库，rollback失效估计要换工作。。。

* 已测试环境
    * Python 2.7,
    * MySQL 5.7


安装
==============
用pip安装
pip install +*.gz
```

使用
=========

### MySQL server必须设置以下参数:

    [mysqld]
    server_id = 1
    log_bin = /var/log/mysql/mysql-bin.log
    max_binlog_size = 1G
    binlog_format = row
    binlog_row_image = full

### user需要的最小权限集合：

    select, super/replication client, replication slave
    
    建议授权
    GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 

**权限说明**

* select：需要读取server端information_schema.COLUMNS表，获取表结构的元信息，拼接成可视化的sql语句
* super/replication client：两个权限都可以，需要执行'SHOW MASTER STATUS', 获取server端的binlog列表
* replication slave：通过BINLOG_DUMP协议获取binlog内容的权限


### 基本用法


**解析出标准SQL**

```bash
shell> python binlog2sql.py -h127.0.0.1 -P3306 -uroot -p'Aa123456' -dtest -t aa id --start-file='mysql-bin.000006'

输出：
INSERT INTO `test`.`test3`(`addtime`, `data`, `id`) VALUES ('2016-12-10 13:03:38', 'english', 4); #start 570 end 736
UPDATE `test`.`test3` SET `addtime`='2016-12-10 12:00:00', `data`='中文', `id`=3 WHERE `addtime`='2016-12-10 13:03:22' AND `data`='中文' AND `id`=3 LIMIT 1; #start 763 end 954
DELETE FROM `test`.`test3` WHERE `addtime`='2016-12-10 13:03:38' AND `data`='english' AND `id`=4 LIMIT 1; #start 981 end 1147
```

**解析出回滚SQL**

```bash

shell> python binlog2sql.py --flashback -h127.0.0.1 -P3306 -uroot -p'Aa123456' -dtest -t aa id --start-file='mysql-bin.000002' --start-position=763 --stop-position=1147

输出：
INSERT INTO `test`.`test3`(`addtime`, `data`, `id`) VALUES ('2016-12-10 13:03:38', 'english', 4); #start 981 end 1147
UPDATE `test`.`test3` SET `addtime`='2016-12-10 13:03:22', `data`='中文', `id`=3 WHERE `addtime`='2016-12-10 12:00:00' AND `data`='中文' AND `id`=3 LIMIT 1; #start 763 end 954
```

### 选项

**mysql连接配置**

-h host; -P port; -u user; -p password

**解析模式**

--stop-never 持续解析binlog。可选。，默认False，同步至执行命令时最新的binlog位置。

-K, --no-primary-key 对INSERT语句去除主键。可选。默认False

-B, --flashback 生成回滚SQL，可解析大文件，不受内存限制。可选。默认False。与stop-never或no-primary-key不能同时添加。

--back-interval -B模式下，每打印一千行回滚SQL，加一句SLEEP多少秒，如不想加SLEEP，请设为0。可选。默认1.0。

**解析范围控制**

--start-file 起始解析文件，只需文件名，无需全路径 。必须。

--start-position/--start-pos 起始解析位置。可选。默认为start-file的起始位置。

--stop-file/--end-file 终止解析文件。可选。默认为start-file同一个文件。若解析模式为stop-never，此选项失效。

--stop-position/--end-pos 终止解析位置。可选。默认为stop-file的最末位置；若解析模式为stop-never，此选项失效。

--start-datetime 起始解析时间，格式'%Y-%m-%d %H:%M:%S'。可选。默认不过滤。

--stop-datetime 终止解析时间，格式'%Y-%m-%d %H:%M:%S'。可选。默认不过滤。

**对象过滤**

-d, --databases 只解析目标db的sql，多个库用空格隔开，如-d db1 db2。可选。默认为空。

-t, --tables 只解析目标table的sql，多张表用空格隔开，如-t user1 user2。可选。默认为空。

--only-dml 只解析dml，忽略ddl。可选。默认TRUE。

--sql-type 只解析指定类型，支持INSERT, UPDATE, DELETE。多个类型用空格隔开，如--sql-type INSERT DELETE。可选。默认为增删改都解析。用了此参数但没填任何类型，则三者都不解析。

### 应用案例

#### **误删整张表数据，需要紧急回滚**

闪回详细介绍可参见example目录下《闪回原理与实战》[example/mysql-flashback-priciple-and-practice.md](./example/mysql-flashback-priciple-and-practice.md)

```bash
test库user表原有数据
mysql> select * from user;
+----+--------+---------------------+
| id | name   | addtime             |
+----+--------+---------------------+
|  1 | 小赵   | 2016-12-10 00:04:33 |
|  2 | 小钱   | 2016-12-10 00:04:48 |
|  3 | 小孙   | 2016-12-13 20:25:00 |
|  4 | 小李   | 2016-12-12 00:00:00 |
+----+--------+---------------------+
4 rows in set (0.00 sec)

mysql> delete from user;
Query OK, 4 rows affected (0.00 sec)

20:28时，user表误操作被清空
mysql> select * from user;
Empty set (0.00 sec)
```

**恢复数据步骤**：

1. 登录mysql，查看目前的binlog文件

	```bash
	mysql> show master status;
	+------------------+-----------+
	| Log_name         | File_size |
	+------------------+-----------+
	| mysql-bin.000051 |       967 |
	| mysql-bin.000052 |       965 |
	+------------------+-----------+
	```

2. 最新的binlog文件是mysql-bin.000052，我们再定位误操作SQL的binlog位置。误操作人只能知道大致的误操作时间，我们根据大致时间过滤数据。

	```bash
	shell> ./binlog2sql.py -h127.0.0.1 -P3306 -uroot -p'Aa123456' -dtest -tuser --start-file='mysql-bin.000007' --start-datetime='2017-12-29 10:50:00' --stop-datetime='2017-12-29 10:57:00'-B > rollback.sql
	输出：
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2013-11-11 00:00:00', 4, 'xiaoli'); 
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2016-11-11 20:25:00', 3, 'xiaosun'); 
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2014-11-11 00:04:48', 2, 'xiaoqian'); 
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2013-11-11 00:04:33', 1, 'xiaozhao');

	```

3. 我们得到了误操作sql的准确位置在728-938之间，再根据位置进一步过滤，使用flashback模式生成回滚sql，检查回滚sql是否正确(注：真实环境下，此步经常会进一步筛选出需要的sql。结合grep、编辑器等)

	```bash
	shell>./binlog2sql.py -h127.0.0.1 -P3306 -uroot -p'Aa123456' -dtest -tuser --start-file='mysql-bin.000007' --start-position=3910 --stop-position=4207 -B > rollback1.sql
	输出：
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2013-11-11 00:00:00', 4, 'xiaoli'); #start 3910 end 4207 time 2017-12-29 10:54:44
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2016-11-11 20:25:00', 3, 'xiaosun'); #start 3910 end 4207 time 2017-12-29 10:54:44
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2014-11-11 00:04:48', 2, 'xiaoqian'); #start 3910 end 4207 time 2017-12-29 10:54:44
	INSERT INTO `test`.`user`(`addtime`, `id`, `name`) VALUES ('2013-11-11 00:04:33', 1, 'xiaozhao'); #start 3910 end 4207 time 2017-12-29 10:54:44

	```

4. 确认回滚sql正确，执行回滚语句。登录mysql确认，数据回滚成功。

	```bash
	shell> mysql -h127.0.0.1 -P3306 -uroot -p'Aa123456' < rollback1.sql

	mysql> select * from user;
	+----+--------+---------------------+
	| id | name   | addtime             |
	+----+--------+---------------------+
	|  1 | 小赵   | 2016-12-10 00:04:33 |
	|  2 | 小钱   | 2016-12-10 00:04:48 |
	|  3 | 小孙   | 2016-12-13 20:25:00 |
	|  4 | 小李   | 2016-12-12 00:00:00 |
	+----+--------+---------------------+
	```

### 限制（对比mysqlbinlog）

* mysql server必须开启，离线模式下不能解析
* 参数 _binlog\_row\_image_ 必须为FULL，暂不支持MINIMAL
* 解析速度不如mysqlbinlog

### 优点（对比mysqlbinlog）

* 纯Python开发，安装与使用都很简单
* 自带flashback、no-primary-key解析模式，无需再装补丁
* flashback模式下，更适合闪回[实战](./example/mysql-flashback-priciple-and-practice.md)
* 解析为标准SQL，方便理解、筛选
* 代码容易改造，可以支持更多个性化解析


