
Schema与数据类型优化


4.1选择优化的数据类型

选择原则:
更小的通常更好:更小的通常更快
简单就好:eg 整型比字符的操作代价更低
尽量避免NULL:因为可为NULL使得索引,索引统计和值比较都更复杂.占用更多的存储空间


不同场景推荐的类型:
存储日期和时间:使用MySQL内建的类型(date,time,datetime),不适用字符串
datetime存储日期加时间,YYYYMMDDHHMMSS
date存储日期,格式:  9999-12-31
time存储时间,格式:  HH:MM:SS
mysql5.7之后,date类型存储日期,优点:
	1.date只需要3个字节,int需要4个字节
	2.date类型可以利用日期时间函数进行日期之间的计算
	3.保存范围 1000-01-01到9999-12-31之间的日期
	
存储IP地址:使用整型
通常情况最好指定列为NOT NULL,除非真的需要存储NULL值
存储财务数据:binint(将需要存储的货币单位,根据小数的位数乘以响应的倍数即可)
密码MD5值: char
存储比秒更小的粒度,可以使用bigint

4.1.1整数类型
tinyint,smallint,mediumint,int,bigint
1个字节8位
分别存储空间:8,16,24,32,64位.eg.tinyint,8位存储范围2^8 = 256-1(整数最大值)

unsigned属性:不允许负值,整数提高一倍上限

指定宽度.eg int(11) (对大多数应用是没意义的,不会限制值的合法范围,只规定了显示字符的个数)
对于存储和计算,int(1)和int(20)是相同的.

4.1.2实数类型
decimal:存储精确的小数  占用9个字节
eg.decimal(18,9)小数点两边将各存储9个数字,一共9个字节,数字分别占4个字节,小数点占1个字节
decimal(13,2)		//小数2位,整数13位

4.1.3字符串类型
varchar: 适合,字符串列的最大长度比平均长度大很多,字段更新少的
	varchar长度的选择:使用最小的符合需求的长度
char: 适合,存储短字符串 或者 所有值都接近同一个长度  eg.密码的MDT值

blob: tinyblob,smallblob,blob,mediumblob,longblob  (存储的是二进制类型,没有排序规则或字符集)
text: tinytext,smalltext,text,mediumtext,longtext  (有排序规则或字符集)

4.1.4日期和时间类型
MySQL存储的最小时间粒度为秒,存储比秒更小的粒度,可以使用bigint
datetime: 占用8个字节,保存大范围的值,从1001年到9999年,精度为秒,与时区无关,封装到格式为YYYYMMDDHHMMSS的整数中, 特点:可排序
timestamp: 保存从1970年1月1日以来的秒数.和UNIX时间戳相同,范围1970年到2038年
timestamp显示的值依赖于时区.MySQL服务器,操作系统,客户端连接都有时区设置.

时间戳转换为日期: FROM_UNIXTIME()
日期转换为时间戳: UNIX_TIMESTAMP()


4.1.5位数据类型

4.2 MySQL schema 设计中的陷阱

4.3 范式和反范式

反范式优点:
反范式化的schema因为所有数据都在一张表中,可以很好的避免关联

范式化的优点:
范式化的更新操作通常比反范式化要快
当数据较好地范式化时,就只有很少或者没有重复数据,所以只需要修改更少的数据
范式化的表通常更小,可以更好地放在内存里,所以执行操作会更快

实际使用:
混用范式化和反范式化,根据实际情况,例如可以有适当的字段冗余.


4.4缓存表和汇总表
有时提升性能最好的方法是在同一张表中保存眼神的冗余数据,然而,有时也需要创建一张完全独立的汇总表或者缓存表(特别
是为满足检索的需求时).如果能容许少量的脏数据,这是非常好的方法.

缓存表和汇总表,术语没有标准的含义

缓存表:表示存储那些可以比较简单地从schema其他表获取(但是每次获取的速度比较慢)数据的表.(例如.逻辑上冗余的数据)
汇总表:保存的是使用group by语句聚合数据的表.例如,假设需要计算之前24小时内发送的消息数,在一个很繁忙的网站不可能维护一个
实时精确的计数器.作为替代方案,可以每小时生成一张汇总表.缺点是计数器不是100%精确的.

实时计算统计值是很昂贵的操作,因为要么需要扫描表中的大部分数据,要么查询语句可能在某些特定的索引上才能有效运行.

在使用缓存表和汇总表时,必须决定是实时维护数据还是定期重建.哪个更好依赖于应用程序.

当重建汇总表和缓存表时,通常需要保证数据在操作时依然可用.着就需要通过使用"影子表"来实现,
"影子表"指的是一张在真实表"背后"创建的表.例如,如果需要重建my_summary,可以先创建my_summary_new,然后填充好数据,最后和真实表切换

drop table if exists my_summary_new,my_summary_old;
create table my_summary_new like my_summary;
------填充数据-------
rename table my_summary to my_summary_old,my_summary_new to my_summary;
------这样,每次切换版本的时候,可以保留旧版本的数据,如果新表有问题,可以很容易进行快速回滚操作.

4.4.1 物化视图
物化视图实际上是预先计算并且存储在磁盘上的表,可以通过各种各样的策略刷新和更新.
Mysql并不原生支持物化视图,然而使用开源工具Flexviews也可以自己实现物化视图

4.4.2 计数器表
如果应用在表中保存计数器,则在更新计数器时可能碰到并发问题.
计数器表在web应用中很常见,可以用这种表缓存一个用户的朋友数,文件下载次数等.

创建一张独立的表存储计数器:
可以使计数器表小且快
帮助避免查询缓存失效

案例:记录网站的点击次数
create table hit_counter (
 cnt int unsigned not null
 ) engine=InnoDB;
 
 表中只有一行数据,每次点击都会导致计数器进行更新:
 update hit_counter set cnt = cnt + 1;
 
 如果高并发,可以如下操作:
 
 修改表:
create table hit_counter (
 slot tinyint unsigned not null primary key,
 cnt int unsigned not null
 ) engine=InnoDB;

预先在表中添加100行数据
更新点击数时,选择一个随机随机行(slot)更新:
update hit_counter set cnt = cnt+1 where slot = rand() * 100;

统计结果:
select sum(cnt) from hit_counter;



4.5加快alter table 操作的速度

一般而言,大部分alter table 操作将导致MySQL服务中断.

对常见场景,可以使用的2种技巧:
1.先在一台不提供服务的机器上执行alter table 操作,然后和提供服务的主库进行切换.
2."影子拷贝",用要求的表结构创建一张和源表无关的新表,然后通过重命名和删表操作交换两张表.

有2种方法可以改变或者删除一个列的默认值
eg.假如要修改电影的默认租赁期限,从3天改到5天
1.慢的方式:
alter talbe sakila.film MODIFY COLUMN rental_duration TINYINT(3) NOT NULL DEFAULT 5;(MODIFY COLUMN会导致表重建)

2.块的方式:
alter talbe sakila.film ALTER COLUMN rental_duration SET DEFAULT 5;
(这个语句会直接修改.frm文件,不涉及表数据.所以,这个操作非常快)










