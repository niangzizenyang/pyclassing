#sqoop
##1.apache Sqoop
	1. Sqoop 是 Hadoop 和关系数据库服务器之间传送数据的一种工具,可以将关系型数据库导中的数据导入到hdfs上,也可以将hdfs上的数据导入到关系型数据库中(仅仅是工具)

![](https://i.imgur.com/bZ5o0q7.png)
##2.安装
	1. 测试环境:centos 6.7
	2. sqoop版本:1.4.6
	2. 安装环境准备:java,hadoop
	3. 配置文件修改
		1. cd ${SOOP_HOME}/conf
		2. mv sqoop-env.template.sh sqoop-env.sh
		3. vi sqoop-env.sh
			1. export HADOOP_COMMON_HOME=/root/apps/hadoop/
			2. export HADOOP_MAPRED_HOME=/root/apps/hadoop/
			3. export HIVE_HOME=/root/apps/hive
	4. 加入mysql的驱动包
		1. cp /hive/lib/mysql-connector-java-5.1.28.jar	$SQOOP_HOME/lib/
	5. 验证启动
		1. bin/sqoop list-databases --connect jdbc:mysql://localhost:3306/ --username root --password niangzi(命令会加载所有的mysql数据库)
	
![](https://i.imgur.com/0ne7QSZ.png)


##3.导入
	###1. sqoop将单个表从rdbms到hdfs,表中所有数据都对应hdfs的一条记录,表中所有数据都存储为文本文件的文本数据(或者avro,sequence等二进制数据)
	###2. 测试:
		
		DROP TABLE IF EXISTS emp;
		CREATE TABLE emp (
		  id int(11) DEFAULT NULL,
		  name varchar(100) DEFAULT NULL,
		  deg varchar(100) DEFAULT NULL,
		  salary int(11) DEFAULT NULL,
		  dept varchar(10) DEFAULT NULL
		);
		

		INSERT INTO emp VALUES ('1201', 'gopal', 'manager', '50000', 'TP');
		INSERT INTO emp VALUES ('1202', 'manisha', 'Proof reader', '50000', 'TP');
		INSERT INTO emp VALUES ('1203', 'khalil', 'php dev', '30000', 'AC');
		INSERT INTO emp VALUES ('1204', 'prasanth', 'php dev', '30000', 'AC');
		INSERT INTO emp VALUES ('1205', 'kranthi', 'admin', '20000', 'TP');
		

		DROP TABLE IF EXISTS emp_add;
		CREATE TABLE emp_add (
		  id int(11) DEFAULT NULL,
		  hno varchar(100) DEFAULT NULL,
		  street varchar(100) DEFAULT NULL,
		  city varchar(100) DEFAULT NULL
		);
		
		
		INSERT INTO emp_add VALUES ('1201', '288A', 'vgiri', 'jublee');
		INSERT INTO emp_add VALUES ('1202', '108I', 'aoc', 'sec-bad');
		INSERT INTO emp_add VALUES ('1203', '144Z', 'pgutta', 'hyd');
		INSERT INTO emp_add VALUES ('1204', '78B', 'old city', 'sec-bad');
		INSERT INTO emp_add VALUES ('1205', '720X', 'hitec', 'sec-bad');
		
	
		DROP TABLE IF EXISTS emp_conn;
		CREATE TABLE emp_conn (
		  id int(100) DEFAULT NULL,
		  phno varchar(100) DEFAULT NULL,
		  email varchar(100) DEFAULT NULL
		);
		
	
		INSERT INTO emp_conn VALUES ('1201', '2356742', 'gopal@tp.com');
		INSERT INTO emp_conn VALUES ('1202', '1661663', 'manisha@tp.com');
		INSERT INTO emp_conn VALUES ('1203', '8887776', 'khalil@ac.com');
		INSERT INTO emp_conn VALUES ('1204', '9988774', 'prasanth@ac.com');
		INSERT INTO emp_conn VALUES ('1205', '1231231', 'kranthi@tp.com');





	###3. 导入mysql数据到hdfs上
		bin/sqoop import \
		--connect jdbc:mysql://localhost:3306/userdb \
		--username root \
		--password niangzi \
		--target-dir /test/mysqlImportData \ (目标数据导入位置)
		--table emp --m 1
	hdfs dfs -cat /test/mysqlImportData/part-m-00000
![](https://i.imgur.com/tZw9ale.png)

	####4.导入mysql表导入到hive表中
		
	1. 复制表结构
		bin/sqoop create-hive-table \
		--connect jdbc:mysql://node-1:3306/userdb \
		--table emp_add \(mysql表)
		--username root \
		--password niangzi \
		--hive-table emp_add_sp (hive表)
	


![](https://i.imgur.com/4TJVuwI.png)
	  
   
    2.复制表数据

	bin/sqoop import \
	--connect jdbc:mysql://node-1:3306/userdb \
	--username root \
	--password niangzi \
	--table emp_add \
	--hive-table emp_add_sp \
	--hive-import \
	--m 1



![](https://i.imgur.com/ol4CQ6p.png)
	
	###5.导入表子集
	--where 可以指定从关系数据库导入数据时的查询条件。它执行在各自的数据库服务器相应的 SQL 查询，并将结果存储在 HDFS 的目标目录
		
	bin/sqoop import \
	--connect jdbc:mysql://node-1:3306/userdb \
	--username root \
	--password niangzi \
	--where "city ='sec-bad'" \
	--target-dir /query \
	--table emp_add --m 1




![](https://i.imgur.com/Pd6z3EQ.png)

![](https://i.imgur.com/UGaq8yz.png)











	复杂查询条件
	如果你想通过并行的方式导入结果，每个map task需要执行sql查询语句的副本，结果会根据sqoop推测的边界条件分区。	
	query必须包含$CONDITIONS。这样每个scoop程序都会被替换为一个独立的条件。同时你必须指定--split-by.分区
	bin/sqoop import \
	--connect jdbc:mysql://node-1:3306/userdb \
	--username root \
	--password niangzi \
	--target-dir /wherequery12 \
	--query 'select id,name,deg from emp WHERE id>1203 and $CONDITIONS' \($CONDITIONS 必须添加)
	--split-by id \
	--fields-terminated-by '\t' \
	--m 1

![](https://i.imgur.com/rA3eOHn.png)


![](https://i.imgur.com/9EbMP36.png)
		
	
	
	###6 增量导入
	-check-column (col) 用来作为判断的列名，如 id
	--incremental (mode) append：追加，比如对大于 last-value 指定的值之后的记录进行追加导入。 lastmodified：最后的修改时间，追加 last-value指定的日期之后的记录
	--last-value (value) 指定自从上次导入后列的最大值（大于该指定的值），也可以自己设定某一值
	假设新添加的数据转换成 emp 表如下：
	1206, satish p, grp des, 20000, GR
	下面的命令用于在 EMP 表执行增量导入：
	bin/sqoop import \
	--connect jdbc:mysql://node-1:3306/userdb \
	--username root \
	--password niangzi \
	--table emp --m 1 \
	--incremental append \
	--check-column id \
	--last-value 1205

##4. 导出
		
	##数据从 HDFS 导出到 RDBMS 数据库导出前， 目标表必须存在于目标数据库中。
	默认操作是从将文件中的数据使用 INSERT 语句插入到表中，更新模式下，是生成 UPDATE语句更新表数据。
	以下是 export 命令语法：$ sqoop export (generic-args) (export-arg)

	##1.从hdfs上导出到mysql中
    