
MySQL基准测试

什么是基准测试:
基准测试是一种测量和评估软件性能指标的活动,用于简历某个时刻的性能基准,以便当系统发生软硬件变化
时重新进行基准测试以评估变化对性能的影响.

基准测试的目的:
	1.建立MySQL服务器的性能基准线
	2.模拟比当前系统更高的负载,找出系统的扩展瓶颈
	3.测试不同的硬件,软件,操作系统配置
	4.证明新的硬件设备是否配置正确

2.2基准测试的策略
主要两种策略:
1.针对整个系统的整体测试(集成式full-stack)
	优点:
		能测试整个系统的性能,包括web服务器缓存,数据库等
		能反映出系统中各个组件接口间的性能问题,提现真实性能状况
	缺点:测试设计复杂,消耗时间长
2.单独测试MySQL(单组件式single-component)



选择集成式测试的原因:
1.用户不进关注MySQL本身性能,而是应用整体性能
2.MySQL并非总是应用的瓶颈,通过整体测试可以揭示这一点


基于以下情况,项目初期,可以只做Mysql测试:
1.需要比较不同的schema或查询的性能
2.针对应用中某个具体问题的测试
3.避免漫长的基准测试,可以做短期的基准测试,快速的"周期循环"来测试出某些调整后的效果

如果可能,可以采用生产环境的数据快照做测试

测试什么指标
根据需求测试:
1.吞吐量:常用的测试单位是每秒事务数(TPS)
2.响应时间或者延迟:测试任务所需的整体时间.通常用百分比响应时间来替代最大响应时间.eg.95%的响应时间为5s,
3.并发性:同时工作中的线程数或者连接数
4.可扩展性:给系统增加1倍的工作,在理想情况下就能获得两倍的结果

设计和规划基准测试:
第一步.提出问题并明确目标,然后决定是采用标准基准测试,还是设计专用的测试
		对整个系统还是某一组件
		使用什么样的数据(例如可以用生产环境的备份)
第二步.针对数据运行查询
		准备基准测试及数据收集脚本
		CPU使用率,IO,网络流量,状态,计时器信息等
		脚本:Get_Test_info.sh   //见目录先文件
		
		运行基准测试
		
		保存及分析基准测试结果
		分析脚本:analyze.sh

2.4基准测试工具

1.集成式测试工具:
ab : 
是Apache HTTP服务器基准测试工具.测试HTTP服务器每秒最多可以处理多少请求.(只能针对单个URL进行尽可能快的压力测试)

http_load:
与ab类似,比ab更加灵活

JMeter:
JMeter是一个Java应用程序,可以加载其他应用并测试其性能.

2.单组件测试工具:
mysqlslap:(mysql自带,不需要安装)
可以模拟服务器的负载,并输出计时信息.包含在MySQL中,测试是可以执行并发连接数,指定SQL语句
可以指定也可以自动生成查询语句
常用参数说明:
--auto-generate-sql	//由系统自动生成SQL脚本进行测试
--auto-generate-sql-add-autoincrement	//在生成的表中增加自增ID
--auto-generate-sql-load-type			//指定测试中使用的查询类型
--auto-generate-sql-write-number		//指定初始化数据时生成的数据量
--concurrency							//指定并发线程的数量
--engine	//指定要测试表的存储引擎,可以用逗号分割多个存储引擎
--no-drop	//指定不清理测试数据
--iterations	//指定测试运行的次数
--number-of-queries	//指定每一个线程执行的查询数量
--number-int-cols	//指定测试表中包含的INT类型列的数量
--number-char-cols	//指定测试表中包含的varchar类型列的数量
--create-schema		//指定了用于执行测试的数据库的名字
--query				//指定自定义的SQL的脚本
--only-print		//并不运行测试脚本,而是把生成的脚本打印出来

eg.
mysqlslap --help		//查看系统是否安装了这个工具
执行测试:
mysqlslap --concurrency=1,50,100,200 --iterations=3 --number-int-cols=5 --number-char-cols=5 --auto-generate-sql --engine=innodb --number-of-queries=10 --create-schema=数据库名字

mysqlslap --concurrency=1,50,100,200 --iterations=3 --number-int-cols=5 --number-char-cols=5 --auto-generate-sql --engine=innodb --number-of-queries=10 --create-schema=qyh


MySQL Benchmark Suite(sql-bench)
包含在MySQL中,单线程,主要用于测试服务器执行查询的速度

Super Smack:
用户MySQL和PostgreSQL的基准测试工具.可以提供压力测试和负载生成.可以模拟多用户访问,支持使用随机数据填充测试表.


Mysql基准测试工具:sysbench
能测试影响服务器各种性能的因素
安装说明:
https://github.com/akopytov/sysbench
# unzip sysbench-0.5.zip











