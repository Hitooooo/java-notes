---
title: sql必知必会
date: 2020-04-10 08:41:46
tags:
- sql
---

#  MySQL必知必会

一、查询数据

### 1. SELECT语句

#### 1.1 检索单个列

```mysql
SELECT prod_name FROM products;
```

**注意：MySQL是不区分大小写的，包括关键字和查询字段**
<!--more-->
#### 1.2 检索多个列

```mysql
SELECT prod_name, prod_price FROM products;
```

检索所有列：

```mysql
SELECT * FROM products;
```

**最好不要使用  *，除非你真的是希望查询所有的字段**

#### 1.3 限制检索结果

为了返回检索结果的第一行或前几行，可以通过LIMIT子句限制结果。

```mysql
SELECT prod_name FROM products LIMIT 5; # 取出结果的前五个数据
```

当然也可以指定 **开始行和行数：**

```mysql
SELECT prod_name FROM products LIMIT 5, 5;# 从第五行开始，需要五个数据 index 5-->9
```

**注意：MySQL起始行是0，LIMIT 1, 1指的是第二行！**

#### 1.4 排序检索结果

> ORDER BY 子句，根据需要排序检出数据。

```SELETC prod_name FROM products;```其实,检出的数据并不是完全随机的,而是按照底层表中出现的顺序显示.

* ASC: 升序 没必要设置,因为这是缺省值
* DESC: 降序 (从Z到A)

```mysql
SELECT prod_id, prod_name FROM products ORDER BY prod_price DESC; # 按价格降序排列
```

当然也是可以多列参与排序，而且可以为**不同列指定不同的排序顺序**

```mysql
SELECT prod_id, prod_name FROM products ORDER BY prod_price DESC, prod_name; # 按价格降序排列
```

> 区分大小写和排序顺序：A与a相同吗？是的，在字典排序中，A被视为与a相同。如果你想更改这种排序，需要请求DBA。

#### 1.5 过滤数据

使用 **WHERE**子句可以指定搜索条件（search criteria）。

操作符如下

| 操作符  | 说明             |
| ------- | ---------------- |
| =       | 等于             |
| <>      | 不等于           |
| !=      | 不等于           |
| <       | 小于             |
| >       | 大于             |
| <=      | 小于等于         |
| \>=     | 大于等于         |
| BETWEEN | 指定的两个值之间 |

##### 1.5.1 检查单个值

```mysql
SELECT prod_name FROM products WHERE prod_name = 'fues';# mysql不区分大小写, Fues的行同样会返回
```

##### 1.5.2 范围检查

```mysql
SELECT prod_name FROM products WHERE prod_price BETWEEN 5 AND 10; # 价格在5到10之间的所有产品

```

##### 1.5.3 控制查询

如果一个列不包含值，称其为 **NULL**。注意NULL与空是不用的概念的，```''```是空但不是null。

```mysql
SELECT prod_name FROM products WHERE prod_name IS NULL; # 返回没有名称的所有产品，不包括 ''

```

##### 1.5.4 OR操作符

```mysql
SELECT prod_name, prod_price FROM products WHERE vend_id = 1002 OR vend_id = 1003;

```

> WHERE可包含任意数目的AND和OR操作符，但是不同的写法可能会造成不同的计算顺序



例如：希望找到价格为8.99和5.99以及价格在10以上的产品

```mysql
SELECT prod_name, prod_price FROM Products WHERE prod_price = 8.99 OR prod_price = 5.99 AND prod_price >=10;

```

可结果是如下，并没有价格超过10的，因为计算次序问题。**AND 操作符的优先级大于 OR，所以注意加括号**

prod_name	prod_price
12 inch teddy bear	8.99

##### 1.5.5 IN 操作符

```mysql
SELECT XX FROM yy WHERE zz IN (A,B,C);

```

IN操作符与OR是类似的，但是IN更加清晰直观。

#### 1.6 用通配符进行过滤

##### 1.6.1 LIKE操作符

* 最常使用的通配符是 **百分号%**,表示任何字符出现任意次数。

  ```mysql
  SELECT prod_id FROM products WHERE prod_name LIKE "jet%"; # 找到以jet开头的产品名称的ID
  
  ```

  > 如果MYSQL配置了区分大小写（建表时，字段增加BINARY 属性），那么搜索也是匹配大小写的。

* 下划线（_）通配符：匹配单个字符

  ```mysql
  SELECT prod_id FROM products WHERE prod_name LIKE '_ ton anvil';
  
  ```

> 正如所见，通配符很有用，但是搜索时间更长
>
> * 不要过度使用
> * 除非绝对必要，否则不要把模糊搜索放在开始处
> * 仔细注意通配符的位置，如果放错位置可能不会返回想要的结果

#### 1.7 使用正则表达式搜索

[菜鸟教程](https://www.runoob.com/mysql/mysql-regexp.html)

### 2. 创建计算字段

很多时候，直接查询出来的数据结果是需要处理后返回的，虽然这个操作也可以交给应用层来做，但是DBS会让APP更加简洁。

#### 2.1 拼接字段

**Concat()将值连结到一起构成单个值。**

```mysql
SELECT CONCAT(prod_id,"_",prod_desc) FROM Products WHERE prod_name LIKE '%ueen 	doll';

```

#### 2.2 去除空格

**Trim() 剪切空格，还可以单边剪切 LTrim(),RTrim()**

#### 2.3 执行算术计算

MySQL支持下表中的基本算术操作符

| 操作符 | 说明 |
| ------ | ---- |
| +      | 加   |
| -      | 减   |
| *      | 乘   |
| /      | 除   |

```mysql
SELECT quantity * item_price AS expanded_price FROM OrderItems WHERE order_num = 20005;

```

### 3. 数据处理函数

#### 3.1 字符串处理工具

| 函数     | 说明                 |
| -------- | -------------------- |
| Left()   | 返回字符串左边的字符 |
| Length() | 返回字符串的长度     |
| Locate() | 超出字符串的子串     |
| Lower()  | 转换为小写           |

![](C:\Users\HitoM\Desktop\1113510-20181226155910408-1414424975.png)

#### 3.2 日期和时间处理函数

首先应该注意的是MySQL使用的日期格式，强烈推荐使用 **yyyy-mm-dd **的格式，其他日期格式可能也行，但是这是首选的。

```mysql
select cust_id ,order_num FROM Orders WHERE order_date = '2004-01-12';

```

**如果日期后由具体时间，而且并不是默认的00：00：00,你会发现上面这种写法是查询不到的，那么你就需要日期处理函数Date()**

```mysql
select cust_id ,order_num FROM Orders WHERE Date(order_date) = '2004-01-12';

```

| 函数                                                         | 描述                                |
| :----------------------------------------------------------- | :---------------------------------- |
| [NOW()](https://www.w3school.com.cn/sql/func_now.asp)        | 返回当前的日期和时间                |
| [CURDATE()](https://www.w3school.com.cn/sql/func_curdate.asp) | 返回当前的日期                      |
| [CURTIME()](https://www.w3school.com.cn/sql/func_curtime.asp) | 返回当前的时间                      |
| [DATE()](https://www.w3school.com.cn/sql/func_date.asp)      | 提取日期或日期/时间表达式的日期部分 |
| [EXTRACT()](https://www.w3school.com.cn/sql/func_extract.asp) | 返回日期/时间按的单独部分           |
| [DATE_ADD()](https://www.w3school.com.cn/sql/func_date_add.asp) | 给日期添加指定的时间间隔            |
| [DATE_SUB()](https://www.w3school.com.cn/sql/func_date_sub.asp) | 从日期减去指定的时间间隔            |
| [DATEDIFF()](https://www.w3school.com.cn/sql/func_datediff_mysql.asp) | 返回两个日期之间的天数              |
| [DATE_FORMAT()](https://www.w3school.com.cn/sql/func_date_format.asp) | 用不同的格式显示日期/时间           |

#### 3.3 数值处理函数

| 函数          | 描述                               |
| ------------- | ---------------------------------- |
| ABS(X)        | 求绝对值                           |
| Ceil(x)       | 大于x的最大整数                    |
| Floor(x)      | 小于x的最大整数                    |
| Mod(x,y)      | x/y的模                            |
| Rand()        | 0-1内的随机值                      |
| Rand(x,y)     | 返回参数x的四舍五入的有y位小数的值 |
| Truncate(x,y) | 返回数字x截断为y位小数的结果       |

### 4. 汇总数据

#### 4.1 聚集函数

| 函 数   | 说 明            |
| ------- | ---------------- |
| AVG()   | 返回某列的平均值 |
| COUNT() | 返回某列的行数   |
| MAX()   | 返回某列的最大值 |
| MIN()   | 返回某列的最小值 |
| SUM()   | 返回某列值之和   |

1. AVG()函数

   ```mysql
   SELECT AVG(prod_price) AS avg_price FROM Products;
   
   ```

   > * **只能用于单个列，多列需要多个AVG函数**
   > * **AVG()忽略值为NULL的行**

2. COUNT()函数

   ```mysql
   SELECT COUNT(cust_email) as num_cust FROM Customers;
   
   ```

   > **NULL值** 如果指定列名，则指定列的值为空会被COUNT忽略，但是如果是COUNT(*),则不忽略。 

3. MAX()/MIN() 数值时返回的时列最大值/最小值，文本则返回首条记录/最后一条记录

   > **NULL值** MIN()函数忽略值为NULL的行

4. SUM求和

   > **NULL值** sum()函数忽略值为NULL的行

### 5. 分组查询

```mysql
SELECT vend_id, count(*) AS num_prods FROM Products GROUP BY vend_id;

```

> *  如果分组列中具有NULL值，则NULL将作为一个分组返回。如果列中有多行NULL值，它们将分为一组。
> *  GROUP BY子句必须出现在WHERE子句之后，ORDER BY子句之前。

#### 5.1 过滤分组

```MYSQL
SELECT cust_id, COUNT(*) as orders FROM Orders GROUP BY cust_id HAVING COUNT(*) >= 2;

```

**HAVING与WHERE的区别：**where在数据分组前过滤，HAVING在数据分组后过滤。

### 6. 连结表

关联供应商表和产品表

```mysql
SELECT vend_name, prod_name, prod_price FROM Vendors, Products WHERE Vendors.vend_id=Products.vend_id ORDER BY vend_name, prod_name;

```

#### 6.1 内部连结

基于两个表之间的相等测试，这种连结成为**内部连结**，上面的例子也是。不过有更规范的语法来指明内连结：

```mysql
SELECT vend_name, prod_name, prod_price FROM Vendors INNER JOIN Products ON Vendors.vend_id=Products.vend_id ORDER BY vend_name, prod_name; # xx INNER JOIN yy ON xx.id = yy.id

```

#### 6.2 高级连结

##### 6.2.1 自联结

假如你发现某物品（其ID为DTNTR）存在问题，因此想知道生产该物品的供应商生产的其他物品是否也存在这些问题。此查询要首先找到生产ID为DTNTR的物品的供应商，然后找出这个供应商生产的其他物品。下面是解决此问题的一种方法

```mysql
SELECT prod_id, prod_name FROM products where vend_id=(SELECT vend_id FROM products WHERE prod_id = 'DTNTR');

```

用自联结实现呢？

```mysql
SELECT p1.prod_name, p1.prod_id FROM Products as p1, Products as p2 WHERE p1.vend_id = p2.vend_id AND p2.prod_id = "BR01";

```

这里面时供应商ID做连结，多值连结。连结后的表同一个供应商有 count(vend_id)^2个，通过另一个prod_id即可找到所有同一个供应商提供的产品编号。

##### 6.2.1 自然连结

出现重复列的时候，只保留一列。这就需要我们手动实现。

```mysql
# 保留C中全部，其余手动给出
SELECT c.*, o.order_num, o.cust_id,oi.prod_id,oi.quantity,oi.item_price FROM Customers as c, Orders as o, OrderItems as oi WHERE c.cust_id = o.cust_id and oi.order_num = o.order_num and prod_id = 'BNBG01';

```

##### 6.2.3 外部连结

如果两张表在关联时，存在无法对应的关系，内部关联或等值关联是不会保留无法对应的行，但有时候需要保留没有关联的那些行。

* LEFT OUTER JOIN: 无法匹配保留左表的值，右表对应的值置为NULL
* RIGHT OUTER JOIN:无法匹配保留由表的值，右表对应的值置为NULL

> 存在两种基本的外部联结形式：左外部联结和右外部联结。它们之间的唯一差别是所关联的表的顺序不同。换句话说，左外部联结可通过颠倒FROM或WHERE子句中

###  7. 组合查询

利用UNION操作符可以将多条SELECT语句组合成一个结果集。

```mysql
SELECT vend_id,  prod_price FROM Products WHERE prod_price <= 5
UNION
SELECT vend_id,  prod_price FROM Products WHERE prod_price >= 10;

```

> UNION 会自动去除重复的行, 如果想保留的话,就是用 UNION ALL

### 8. 全文搜索

其实MySQL也是支持全文搜索的,只是InnoDB引擎不支持,而MyISAM是支持的.但是没人用MySQL做搜索,自行了解其他的吧.如ES,Solr

##  二、插入数据（INSERT）

```mysql
INSERT INTO 表名(字段名1, 字段名2,...) VALUES(值1, 值2),(值1, 值2); # 向表中插入俩条数据

```

#### 1. 插入检索出的数据

```mysql
INSERT INTO XX(aa,bb,cc) SELECT aa,bb,cc FROM XXNew Where zz=j;

```

## 三、更新和删除(UPDATE & DROP)

```mysql
UPDATE table_name SET field1=new-value1, field2=new-value2 [WHERE Clause]

```

- 你可以同时更新一个或多个字段
- 你可以在 WHERE 子句中指定任何条件
- 你可以在一个单独表中同时更新数据

```mysql
UPDATE Customers SET cust_zip=8877, cust_name='hito' WHERE cust_state = 'AZ;

```

### 1. 删除

#### 1.1 删除表

```mysql
DROP TABLE table_name ;

```

#### 1.2 删除行

```mysql
DELETE FROM Vendors WHERE vend_id = 89757;

```

注意还有个清空表的操作 ,注意这比一行行删除快，但是不可恢复，因为它是清空表后重建。[具体注意事项见官网](https://dev.mysql.com/doc/refman/8.0/en/truncate-table.html)

```mysql
TRUNCATE TABLE XX;

```

#### 1.3 删除数据库

```mysql
drop database <数据库名>;

```

## 四、创建表

```mysql
CREATE TABLE table_name (column_name column_type);

```

引擎

1. InnoDB：可靠的事务处理引擎
2. MEMORY：功能等同于MyISAM，但是数据存储在内存，速度快，适合临时表
3. MyISAM：高性能的、支持全文搜索的引擎，但不支持事务

```mysql
CREATE TABLE IF NOT EXISTS `runoob_tbl`(
   `runoob_id` INT UNSIGNED AUTO_INCREMENT,
   `runoob_title` VARCHAR(100) NOT NULL,
   `runoob_author` VARCHAR(40) NOT NULL,
   `submission_date` DATE DEFAULT '2019-09-21',
   PRIMARY KEY ( `runoob_id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

- 如果你不想字段为 **NULL** 可以设置字段的属性为 **NOT NULL**， 在操作数据库时如果输入该字段的数据为**NULL** ，就会报错。
- AUTO_INCREMENT定义列为自增的属性，一般用于主键，数值会自动加1。
- PRIMARY KEY关键字用于定义列为主键。 您可以使用多列来定义主键，列间以逗号分隔。
- ENGINE 设置存储引擎，CHARSET 设置编码。

PS: [关于DATETIME和TIMESTAMP](https://www.vertabelo.com/blog/what-datatype-should-you-use-to-represent-time-in-mysql-we-compare-datetime-timestamp-and-int/)

### 1. 修改表

```mysql
ALERT TABLE 表名 ADD COLUMN 字段名 属性...;

```

```mysql
ALERT TABLE 表名 DROP COLUMN 字段名;

```

## 五、视图

将一些复杂的、常用的SQL语句封装起来，其结果可以看作一张“实际”存在的表来操作，这张表就是视图。创建View的语句：

```mysql
CREATE VIEW view_name AS (需要封装的语句);

```

```mysql
CREATE VIEW testView AS
SELECT vend_id,  prod_price FROM Products WHERE prod_price <= 5
UNION
SELECT vend_id,  prod_price FROM Products WHERE prod_price >= 10;
# 只需要查询视图数据即可
SELECT * From testView;

```

**那么直接修改视图的话，原表会得到更新吗？**

* 删除视图：不会删除原表
* 对视图进行更新操作：如果由修改权限，并且更新合法，是可以更新原表的。

## 六、存储过程

存储过程也是一种SQL语句的封装，但是自定义程度更高，接受入参并能返回结果，而且相比较组合执行SQL是语句性能更高。

### 1. 创建存储过程

```mysql
CREATE
    [DEFINER = user]
    PROCEDURE sp_name ([proc_parameter[,...]])
    [characteristic ...] routine_body
# proc_parameter: [ IN | OUT | INOUT ] param_name type

```

```mysql
# 创建，接收一个用来装返回值的参数
CREATE PROCEDURE productpricing2(OUT avg_p DOUBLE)
BEGIN
	SELECT Avg(prod_price) INTO avg_p
	FROM Products;
END;
# 调用存储过程后，查看结果
CALL productpricing2(@lala);
SELECT @lala;

```

你需要获得与以前一样的订单合计，但需要对合计增加营业税，不过只针对某些顾客（或许是你所在州中那些顾客）。那么，
你需要做下面几件事情： 

* 获得合计（与以前一样）； 
* 把营业税有条件地添加到合计； 
* 返回合计（带或不带税）

```mysql
#Name:ordertotal
#Parameters:onumber=order number
#                     taxable=0 if not taxable,1 if taxable
#                     ototal=order total variable
CREATE PROCEDURE ordertotal(
    IN onumber INT,
    IN taxable BOOLEAN,
    OUT ototal DECIMAL(8,2)
)COMMENT 'Obtain order total,optionally adding tax'
BEGIN
    #Declare variable for total
    DECLARE total DECIMAL(8,2);
    #Declare tax percentage
    DECLARE taxrate INT DEFAULT 6;
    #Get the order total
    SELECT Sum(item_price*quantity)
    FROM orderitems
    WHERE order_num=onumber
    INTO total;
    #Is this taxable?
    IF taxable THEN
            #Yes,so add taxrate to the total
        SELECT total+(total/100*taxrate)INTO total;
    END IF;
    # AND finally,save to out variable
    SELECT total INTO ototal;
END;    

```

## 七、使用游标(TODO)

## 八、触发器











