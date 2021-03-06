# JDBC

## 加载数据

JDBC其实是一类数据源，比如MySQL, Oracle,Hive thrift server等等，只要数据源支持JDBC协议，那么就可以
通过JDBC进行加载。在这里，我们会以MySQL为主要例子进行介绍，其他也会提及。

首先我们需要建立连接，这是通过connect语法来完成的：

```sql
 set user="root";
 
 connect jdbc where
 url="jdbc:mysql://127.0.0.1:3306/wow?characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false"
 and driver="com.mysql.jdbc.Driver"
 and user="${user}"
 and password="${password}"
 as db_1;
 
```

这句话表示，我希望连接jdbc数据源，连接相关的参数在where语句里，驱动是MySQL的驱动，然后设置用户名和密码。
我们把我们这个链接叫做db_1,其实就是wow的别名，wow是MySQL里的一个DB.

接着我就可以这个数据库里加载任意表了：

```
load jdbc.`db_1.table1` as table1;
load jdbc.`db_1.table2` as table2;

select * from table1 as output;
```

下面是一些参见参数：

| Property Name  |  Meaning |
|---|---|
|url|The JDBC URL to connect to. The source-specific connection properties may be specified in the URL. e.g., jdbc:postgresql://localhost/test?user=fred&password=secret|
|dbtable |The JDBC table that should be read. Note that anything that is valid in a FROM clause of a SQL query can be used. For example, instead of a full table you could also use a subquery in parentheses.|
|driver |The class name of the JDBC driver to use to connect to this URL.|
|partitionColumn, lowerBound, upperBound|	These options must all be specified if any of them is specified. In addition, numPartitions must be specified. They describe how to partition the table when reading in parallel from multiple workers. partitionColumn must be a numeric column from the table in question. Notice that lowerBound and upperBound are just used to decide the partition stride, not for filtering the rows in table. So all rows in the table will be partitioned and returned. This option applies only to reading.|
|numPartitions|	The maximum number of partitions that can be used for parallelism in table reading and writing. This also determines the maximum number of concurrent JDBC connections. If the number of partitions to write exceeds this limit, we decrease it to this limit by calling coalesce(numPartitions) before writing.|
|fetchsize|	The JDBC fetch size, which determines how many rows to fetch per round trip. This can help performance on JDBC drivers which default to low fetch size (eg. Oracle with 10 rows). This option applies only to reading.|
|batchsize|	The JDBC batch size, which determines how many rows to insert per round trip. This can help performance on JDBC drivers. This option applies only to writing. It defaults to 1000.|
|isolationLevel|	The transaction isolation level, which applies to current connection. It can be one of NONE, READ_COMMITTED, READ_UNCOMMITTED, REPEATABLE_READ, or SERIALIZABLE, corresponding to standard transaction isolation levels defined by JDBC's Connection object, with default of READ_UNCOMMITTED. This option applies only to writing. Please refer the documentation in java.sql.Connection.|
|sessionInitStatement|	After each database session is opened to the remote DB and before starting to read data, this option executes a custom SQL statement (or a PL/SQL block). Use this to implement session initialization code. Example: option("sessionInitStatement", """BEGIN execute immediate 'alter session set "_serial_direct_read"=true'; END;""")|
|truncate|	This is a JDBC writer related option. When SaveMode.Overwrite is enabled, this option causes Spark to truncate an existing table instead of dropping and recreating it. This can be more efficient, and prevents the table metadata (e.g., indices) from being removed. However, it will not work in some cases, such as when the new data has a different schema. It defaults to false. This option applies only to writing.|
|createTableOptions|	This is a JDBC writer related option. If specified, this option allows setting of database-specific table and partition options when creating a table (e.g., CREATE TABLE t (name string) ENGINE=InnoDB.). This option applies only to writing.|
|createTableColumnTypes|	The database column data types to use instead of the defaults, when creating the table. Data type information should be specified in the same format as CREATE TABLE columns syntax (e.g: "name CHAR(64), comments VARCHAR(1024)"). The specified types should be valid spark sql data types. This option applies only to writing.|
|customSchema|	The custom schema to use for reading data from JDBC connectors. For example, "id DECIMAL(38, 0), name STRING". You can also specify partial fields, and the others use the default type mapping. For example, "id DECIMAL(38, 0)". The column names should be identical to the corresponding column names of JDBC table. Users can specify the corresponding data types of Spark SQL instead of using the defaults. This option applies only to reading.|

其中，partitionColumn, lowerBound, upperBound,numPartitions 用来控制加载表的并行度。如果你
加载数据太慢，那么可以调整着几个参数。

当然，我们也可以不用connect语法，直接使用Load语法：

```sql
load jdbc.`db.table` options
and driver="com.mysql.jdbc.Driver"
and url="jdbc:mysql://127.0.0.1:3306/...."
and user="..."
and password="...."
as table1;
```

值得注意的是，JDBC还支持使用MySQL原生SQL的方式去加载MySQL数据。比如：

```sql
load jdbc.`db_1.test1` where directQuery='''
select * from test1 where a = "b"
''' as newtable;

select * from newtable;
```

这种情况要求加载的数据集不能太大。

## 保存更新数据

如同加载数据一样，你可以复用前面的数据连接：

```sql
 set user="root";
 
 connect jdbc where
 url="jdbc:mysql://127.0.0.1:3306/wow?characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false"
 and driver="com.mysql.jdbc.Driver"
 and user="${user}"
 and password="${password}"
 as db_1;
 
```

接着对得到的数据进行保存：

```
save append tmp_article_table as jdbc.`db_1.crawler_table`;
```

这句话表示，我们需要保存，并且使用追加的方式，往 db_1中的 crawler_table里添加数据，这些数据来源于表
tmp_article_table。 如果你需要覆盖请使用 

```
save overwrite ....
```


如果你希望创建表，然后再写入表，那么你可以使用JDBC Transformer,

```
--创建一张无意义的表，因为语法需要
select 1 as a as FAKE_TABLE;

run FAKE_TABLE as JDBC.`db_1` where 
driver-statement-0="drop table test1"
and driver-statement-1="create table test1.....";

save append tmp_article_table as jdbc.`db_1.test1`;
```


这段语句，我们先删除test1,然后创建test1,最后使用save语句把进行数据结果的保存。





