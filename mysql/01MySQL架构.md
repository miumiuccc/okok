
1.1MySQL逻辑架构
第一层:连接处理/线程处理/授权认证/安全认证等
第二层:大多数核心服务功能:eg.查询解析,分析,优化,缓存,内置函数
第三层:存储引擎(负责MySQL中数据的存储和提取)


1.2MySQL如何控制并发读写:
共享锁(shared lock),也叫读锁(read lock)
排他锁(exclusive lock),也叫写锁(write lock)

读锁:共享锁,相互不阻塞,多个用户可以同时读取同一内容.
写锁:排他锁,写锁会阻塞其他的写锁和读锁.

锁粒度:锁定的数据量越少,则系统的并发程度越高,只要相互之间不发生冲突即可.
锁策略:在锁的开销(获得锁,检查锁等都需要开销资源)和数据安全性之间寻求平衡

表锁(table lock):开销最少的锁策略,锁定整张表
1.一个用户在对表进行写操作(插入,删除,更新等)前,需要获取写锁,这会阻塞其他用户对该表的所有读写操作.
2.写锁比读锁有更高的优先级,一个写锁请求可能会被插入到读锁队列前面(读锁不能插入到写锁前面)

行级锁(row lock):开销最大,也最大程度支持并发处理.
1.行级锁自在存储引擎层实现,MySQL层没有实现

1.3事务:
良好的事务系统,必备的标准特征ACID:
原子性(atomicity):一个事务为不可分割的最小工作单位,要么全部提交成功,要么全部失败.
一致性(consistency):
隔离性(isolation):通常来说,一个事务最终提交之前,其他事务读取的内容,为提交前数据
持久性(durability):

隔离级别:较低级别的隔离通常可以执行更高的并发,系统开销也更低
READ UNCOMMITTED(未提交读):事务未完全提交,对其他事务可见,其他事务可以读取部分数据,这被称为脏读(Dirty Read).实际中一般很少使用
READ COMMITTED(提交读):事务完全提交后,对其他事务才可见..这个级别有时也叫不可重复读(nonrepeatable read),因为两次执行同样的查询,可能会得到不一样的结果.
REPEATABLE READ(可重复读):MySQL的默认事务隔离级别.保证在同一个事务中多次读取同样记录的结果是一致的.但也没法避免幻读(Phantom Read).InnoDB存储引擎通过多版本并发控制(MVCC)解决了幻读的问题.
SERIALIZABLE(可串行化):最高的隔离级别.通过事务串行执行,避免了幻读的问题.开销大,实际很少应用.

死锁:两个或者多个事务在同一资源上相互占用,并请求锁定对方占用的资源,从而导致恶性循环的现象.
InnoDB目前处理死锁的方法是,将持有最少行级排他锁的事务进行回滚(这是相对比较简单的死锁回滚算法).

事务自动提交:MySQL默认采用自动提交模式(Autocommit).如果不是显式地开始一个事务,则每个查询都被当做一个事务执行提交操作.
查看自动提交模式:
show variables like 'autocommit';
启动/禁止自动提交:  1:on启动 | 2:off禁止
set autocommit = 1;

MySQL设置隔离级别.InnoDB引擎支持所有的隔离级别.eg.
set transaction isolation level read committed;

如果事务需要回滚,非事务型的表上的变更无法撤销,会导致数据库处于不一致的状态,这种情况很难修复,事务的最终结果将无法确定.这种操作,MySQL通常不会发出提醒,也不会报错.


1.4多版本并发控制

1.5MySQL的存储引擎:
MySQL将每个数据库(也可以称为schema)保存为数据目录下的一个子目录.创建表时,MySQL会在数据库子目录下创建一个和表同名的.frm文件保存表的定义.
eg.查看user表的相关信息
show table status like 'user';

InnoDB存储引擎:
被设计用来处理大量短期(short-lived)事务,短期事务大部分情况是正常提交的,很少会被回滚.
除非有特别的原因需要使用其他的存储引擎,否则应该优先考虑InnoDB引擎.(例如,如果要用到全文索引,建议优先考虑InnoDB+Sphinx的组合,而不是使用支持全文索引的MyISAM)
如果使用了InnoDB,强烈建议阅读官方手册中的"InnoDB事务模型和锁"一节.
如果程序基于InnoDB构建,则事先了解一下InnoDB的MVCC架构带来的一些微妙和细节之处是非常有必要的.
支持行级锁
	1.行级锁可以最大程度支持并发
	2.行级锁是存储引擎层实现的

Innodb使用表空间进行 数据存储
innodb_file_per_table
ON:	独立表空间:表名.ibd	会为每个表创建独立表空间(MySQL5.6及以后默认)
OFF:系统表空间:ibdataX	
eg>
查看设置:
show variables like 'innodb_file_per_table';
设置:
set global innodb_file_per_table=off;

独立表空间&系统表空间的选择:
比较:
	1.系统表空间无法简单的压缩文件大小
	2.独立表空间可以通过optimize table命令压缩系统文件,重建表空间
	3.系统表空间会产生IO瓶颈
	4.独立表空间可以同时向多个文件刷新数据
结论:Innodb使用独立表空间

表转移的步骤:把原来存在系统表空间中的表转移到独立表空间中的方法
1.使用mysqldump导出所有数据库表数据
2.停止MySQL服务,修改参数,并删除Innodb相关文件
3.重启MySQL服务,重建Innodb系统表空间
4.重新导入数据

redo log存储已提交的事务,查看缓冲区大小:
show variables like 'innodb_log_buffer_size';
undo log存储未提交的事务,需要随机读写


MyISAM存储引擎:
不支持事务和行级锁,而且崩溃后无法安全恢复
MyISAM存储引擎表由MYD和MYI组成
支持数据压缩,压缩后不影响读操作,不能再写操作
限制:
版本 > MySQL5.0默认单表支持256TB
检查表:
check table 表名;
修复:
repaire table 表名;
压缩表:(使用这个命令需要关闭mysql服务)
myisampack -b -f 表名.MYI;
适用场景:
非事务型应用
只读类应用
空间类应用(例如gps数据)

CSV存储引擎:
适用场景:
	1.适合作为数据交换的中间表.eg 电子表格 ->CSV文件 ->Mysql数据

Memory存储引擎:
适用场景:
	用于查找或者映射表,例如右边和地区的对应表
	数据存储在缓存中,数据应该可以重建才用这个
	


Archive引擎:
只支持insert和select,每次select查询都需要执行全表扫描.所以Archive表适合日志,数据采集类应用,这类应用做数据分析时往往需要全表扫描.
支持行级锁和专用的缓冲区,所以可以实现高并发的插入.
适用场景:
日志和数据采集类应用(不会对数据进行修改)

选择合适的引擎:
1.优先考虑InnoDB
2.事务:如果不需要事务,并且主要是select和insert操作,可选MyISAM,一般日志型的应用比较符合这一特征
3.备份:如果需要在线热备份,选择InnoDB是基本的要求
4.崩溃恢复:MyISAM崩溃后发生损坏的概率比InnoDB高很多,恢复速度也慢.所以即使不需要事务支持,这个因素考虑的话,很多也选InnoDB.

大数据量:
InnoDB数据库维护的数据量3-5TB之间很不错.
例如10TB以上的级别,可能需要建立数据仓库.Infobright是MySQL数据仓库最成功的解决方案.另外也有TokuDB.


转换表的引擎:
方法1:alter table
这个方法的问题是,需要执行很长的时间.MySQL会按行将数据从原表复制到一张新的表中,在复制期间可能会消耗系统所有的I/O能力,同事原表上会加上读锁.
eg.ALTER TABLE mytable ENGINE = InnoDB;
方法2:导出和导入
使用mysqldump工具将数据导出到文件,修改create table语句的存储引擎选项.
方法3:创建于查询(CREATE和SELECT)
先创建一个新的存储引擎的表,然后利用INSERT...SELECT语法来导数据
eg.
CREATE TABLE innodb_table LIKE myisam_table;
ALTER TABLE innodb_table ENGING = InnoDB;
INSTER INTO innodb_table SELECT * FROM myisam_table;
















