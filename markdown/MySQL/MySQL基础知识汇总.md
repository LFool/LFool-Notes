[toc]

# MySQL

> 本文所有内容均基于文末附上的数据库
>

## 第 1 课：了解 SQL

### <font color=#1FA774>数据库</font>

**<font color='red'>数据库 (database)：</font>**保存有组织的数据的容器（通常是一个文件或一组文件）

**<font color='red'>数据库管理系统 (DBMS)：</font>**DBMS 是用来创建和操作数据库的容器，简称数据库软件

### <font color=#1FA774>表 (table)</font>

表是一种结构化的文件，可用来存储某种特定类型的数据

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**数据库中的每个表都有一个名字来标识自己。这个名字是唯一的，即数据库中没有其他表具有相同的名字

- 使表名成为唯一的，实际上是数据库名和表名等的组合。在同一个数据库中不能存在两个名字相同的表，但在不同的数据库中，是可以存在两个名字相同的表

### <font color=#1FA774>列 (column) 和数据类型</font>

**<font color='red'>列 (column)：</font>**表中的一个字段。所有表都是由一个或多个列组成的

**<font color='red'>数据类型：</font>**允许什么类型的数据。每个表列都有相应的数据类型，它限制 (或允许) 该列中存储的数据

### <font color=#1FA774>行 (row)</font>

**<font color='red'>行 (row)：</font>**表中的一个记录

### <font color=#1FA774>主键 (primary key)</font>

**<font color='red'>主键 (primary key)：</font>**一列(或几列) ,其值能够唯一标识表中每一行

表中的任何列都可以作为主键，只要它满足以下条件：

- 任意两行都不具有相同的主键值
- 每一行都必须具有一个主键值 (主键列不允许空值 NULL)
- 主键列中的值不允许修改或更新
- 主键值不能重用 (如果某行从表中删除，它的主键不能赋给以后的新行)

主键通常定义在表的一列上，但并不是必须这么做，也可以一起使用多个列作为主键。在使用多列作为主键时，上述条件必须应用到所有列，所有列值的组合必须是唯一的

### <font color=#1FA774>SQL 语句</font>

SQL(发音为字母 S-Q-L 或 sequel)是 Structured Query Language (结构化查询语言) 的缩写。SQL 是一种专门用来与数据库沟通的语言

<font color='red'>SQL 语句不区分大小写。习惯：SQL 关键字使用大写；列名 or 表名使用小写</font>

<font color='red'>多条 SQL 语句必须以分号(; )分隔</font>

<font color='red'>在处理 SQL 语句时，其中所有空格都被忽略</font>

<font color='red'>SQL 下标都是从 0 开始</font>

<font color='red'>单引号用来限定字符串，数值不需要</font>

## 第 2 课：检索数据 (SELECT)

**<font color='red'>SELECT 语句：</font>**从一个或多个表中检索信息

```mysql
-- 检索单个列 (数据顺序不定)
SELECT prod_name FROM Products;
-- 检索多个列
SELECT prod_id, prod_name, prod_price FROM Products;
-- 检索所有列
SELECT * FROM Products;

-- 检索不同的值 (检索不同的值)
-- 注意：DISTINCT 关键字作用于所有的列,不仅仅是跟在其后的那一列
SELECT DISTINCT vend_id FROM Products;

-- 限制结果
-- 输出前 5 行
SELECT prod_name FROM Products LIMIT 5;
-- 输出从第 3 行开始 5 行的内容
-- 下面两句等价
SELECT prod_name FROM Products LIMIT 5 OFFSET 3;
SELECT prod_name FROM Products LIMIT 3, 5;
```

**小试牛刀：**

```mysql
-- 编写 SQL 语句，从 Customers 表中检索所有的 ID (cust_id)
SELECT cust_id FROM Customers;
-- OrderItems 表包含了所有已订购的产品(有些已被订购多次)。编写 SQL 语句，检索并列出已订购产品 (prod_id) 的清单 (不用列每个订单，只列出不同产品的清单)
SELECT DISTINCT prod_id FROM OrderItems;
-- 编写 SQL 语句, 检索 Customers 表中所有的列，再编写另外的 SELECT 语句，仅检索顾客的 ID。使用注释，注释掉一条 SELECT 语句，以便运行另一条 SELECT 语句
SELECT * FROM Customers;
SELECT cust_id FROM Customers;
```

## 第 3 课：排序检索数据 (ORDER BY)

`SELECT`检索出来的数据是特定顺序的

**<font color='red'>子句：</font>**SQL 语句由子句构成，有些子句是必需的，有些则是可选的。一个子句通常由一个关键字加上所提供的数据组成

**<font color='red'>ORDER BY 子句：</font>**取一个或多个列的名字，据此对输出进行排序

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**在指定一条 ORDER BY 子句时，应该保证它是 SELECT 语句中最后一条子句

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**如果不指定排序的顺序 (默认升序 ASC)，降序 DESC (只应用到直接位于其前面的列名，如果想在多个列上进行降序排序,必须对每一列指定 DESC 关键字)

```mysql
-- 按单个列排序 (根据 prod_name 排序)
SELECT prod_name FROM Products ORDER BY prod_name;
-- 按多个列排序 (先根据 prod_price 排序，如果 prod_price 相同，再根据 prod_name 排序)
SELECT prod_id, prod_price, prod_name FROM Products ORDER BY prod_price, prod_name;

-- 按列位置排序 (prod_id, prod_price, prod_name 对应 1, 2, 3)
-- 缺点：如果进行排序的列不在 SELECT 清单中，显然不能使用这项技术
SELECT prod_id, prod_price, prod_name FROM Products ORDER BY 2, 3;

-- 指定排序方向 (根据 prod_price 降序排序)
SELECT prod_id, prod_price, prod_name FROM Products ORDER BY prod_price DESC;
-- 先根据 prod_price 降序排序，再根据 prod_name 升序排序
SELECT prod_id, prod_price, prod_name FROM Products ORDER BY prod_price DESC, prod_name;
```

**小试牛刀：**

```mysql
-- 编写 SQL 语句，从 Customers 中检索所有的顾客名称 (cust_name)，并按从 Z 到 A 的顺序显示结果
SELECT cust_name FROM Customers ORDER BY cust_name DESC;
-- 编写 SQL 语句，从 Orders 表中检索顾客 ID (cust_id) 和订单号 (order_num)，并先按顾客 ID 对结果进行排序，再按订单日期倒序排列
SELECT cust_id, order_num FROM Orders ORDER BY cust_id, order_date DESC;
-- 显然，我们的虚拟商店更喜欢出售比较贵的物品，而且这类物品有很多。编写 SQL 语句，显示 OrderItems 表中的数量和价格 (item_price)，并按数量由多到少、价格由高到低排序
SELECT quantity, item_price FROM OrderItems ORDER BY quantity DESC, item_price DESC;
```

## 第 4 课：过滤数据 (WHERE)

数据库表一般包含大量的数据，很少需要检索表中的所有行。通常只会根据特定操作或报告的需要提取表数据的子集。只检索所需数据需要指定搜索条件 (search criteria)，搜索条件也称为过滤条件 (filter condition)

**<font color='red'>WHERE 子句：</font>**数据根据 WHERE 子句中指定的搜索条件进行过滤

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**WHERE 子句在表名 (FROM 子句) 之后给出

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16402516458648251645864825235pN5IJY.png" alt="image-20220226164025008" style="zoom:18%;" /> **总结：**ORDER BY 放最后；WHERE 放 FROM table 后

**<font color='red'>WHERE 子句操作符</font>**

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>** `NULL = no value`，与字段包含 0，空字符串，仅仅包含空格不同

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**通过过滤选择不包含指定值的所有行时, 无法返回含 NULL 值的行

|      操作符       |        说明        |
| :---------------: | :----------------: |
|         =         |        等于        |
|        <>         |       不等于       |
|        ! =        |       不等于       |
|         <         |        小于        |
|        < =        |      小于等于      |
|        !<         |       不小于       |
|         >         |        大于        |
|        > =        |      大于等于      |
|        !>         |       不大于       |
| BETWEEN xx AND xx | 在指定的两个值之间 |
|      IS NULL      |    为 NULL 的值    |

```mysql
-- 检查单个值
SELECT prod_price, prod_name FROM Products WHERE prod_price = 3.49;
-- 检查范围值 (包括 开始值 和 结束值)
-- BETWEEN A AND B => A <= x <= B
SELECT prod_price, prod_name FROM Products WHERE prod_price BETWEEN 5 AND 10;
-- 空值检查
-- 返回所有没有价格的商品
SELECT prod_name FROM Products WHERE prod_price IS NULL;
```

**小试牛刀：**

```mysql
-- 编写 SQL 语句，从 Products 表中检索产品 ID (prod_id) 和产品名称 (prod_name)，只返回价格为 9.49 美元的产品
SELECT prod_id, prod_name FROM Products WHERE prod_price = 9.49;
-- 编写 SQL 语句,从 Products 表中检索产品 ID (prod_id) 和产品名称 (prod_name)，只返回价格为 9 美元或更高的产品
SELECT prod_id, prod_name FROM Products WHERE prod_price >= 9;
-- 结合第 3 课和第 4 课编写 SQL 语句，从 OrderItems 表中检索出所有不同订单号 (order_num)，其中包含 100 个或更多的产品
SELECT DISTINCT order_num FROM OrderItems WHERE quantity >=100;
-- 编写 SQL 语句, 返回 Products 表中所有价格在 3 美元到 6 美元之间的产品的名称 (prod_name) 和价格 (prod_price)，然后按价格对结果进行排序
SELECT prod_name, prod_price FROM products WHERE prod_price BETWEEN 3 AND 6 ORDER BY prod_price;
```

## 第 5 课：高级数据过滤 (AND/OR/IN/NOT)

```mysql
-- AND 操作符
SELECT prod_id, prod_price, prod_name FROM Products WHERE vend_id = 'DLL01' AND prod_price <= 4;

-- OR 操作符
-- 许多 DBMS 在 OR WHERE 子句的第一个条件得到满足的情况下，就不再计算第二个条件了
SELECT prod_id, prod_price, prod_name FROM Products WHERE vend_id = 'DLL01' OR vend_id = 'BRS01';
```

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**在处理 OR 操作符前，优先处理 AND 操作符

```mysql
SELECT prod_name, prod_price FROM Products WHERE vend_id = 'DLL01' OR vend_id = 'BRS01' AND prod_price >= 10;
-- 等价下面的 SQL 语句
SELECT prod_name, prod_price FROM Products WHERE vend_id = 'DLL01' OR (vend_id = 'BRS01' AND prod_price >= 10);
```

**<font color='red'>IN 操作符：</font>**IN 操作符用来指定条件范围，范围中的每个条件都可以进行匹配

**<font color='red'>IN 操作符优点：</font>**

- 在有很多合法选项时，IN 操作符的语法更清楚，更直观
- 在与其他 AND 和 OR 操作符组合使用 IN 时，求值顺序更容易管理
- IN 操作符一般比一组 OR 操作符执行得更快
- IN 的最大优点是可以包含其他 SELECT 语句，能够更动态地建立 WHERE 子句

**<font color='red'>NOT 操作符：</font>**否定其后所跟的任何条件

因为 NOT 从不单独使用 (它总是与其他操作符一起使用)，所以它的语法与其他操作符有所不同。NOT 关键字可以用在要过滤的列前，而不仅是在其后

**<font color='red'>NOT 操作符优点：</font>**与 IN 操作符联合使用时，可`NOT IN (xx, xxx, xxx)`

```mysql
-- IN 操作符
SELECT prod_price, prod_name FROM Products WHERE vend_id IN ('DLL01', 'BRS01');
-- 等价下面 SQL 语句
SELECT prod_price, prod_name FROM Products WHERE vend_id = 'DLL01' OR vend_id = 'BRS01';

-- 列出除 DLL01 之外的所有供应商制造的产品
SELECT prod_name FROM Products WHERE NOT vend_id = 'DLL01' ORDER BY prod_name;
-- -- 等价下面的 SQL 语句
SELECT prod_name FROM Products WHERE vend_id <> 'DLL01' ORDER BY prod_name;
```

**小试牛刀：**

```mysql
-- 编写 SQL 语句，从 Vendors 表中检索供应商名称 (vend_name)，仅返回加利福尼亚州的供应商 (这需要按国家 [USA] 和州 [CA] 进行过滤，没准其他国家也存在一个加利福尼亚州)。提示：过滤器需要匹配字符串
SELECT vend_name FROM Vendors WHERE vend_country = 'USA' AND vend_state = 'CA';
-- 编写 SQL 语句，查找所有至少订购了总量 100 个的 BR01、BR02 或 BR03 的订单。你需要返回 OrderItems 表的订单号 (order_num)、产品 ID(prod_id) 和数量，并按产品 ID 和数量进行过滤。提示：根据编写过滤器的方式，可能需要特别注意求值顺序
-- Solution 1
SELECT order_num, prod_id, quantity FROM OrderItems WHERE (prod_id='BR01' OR prod_id='BR02' OR prod_id='BR03') AND quantity >=100;
-- Solution 2
SELECT order_num, prod_id, quantity FROM OrderItems WHERE prod_id IN ('BR01','BR02','BR03') AND quantity >=100;
-- 现在，我们回顾上一课的挑战题。编写 SQL 语句，返回所有价格在 3 美元到 6 美元之间的产品的名称 (prod_name) 和价格 (prod_price)。使用 AND，然后按价格对结果进行排序
SELECT prod_name, prod_price FROM products WHERE prod_price >= 3 AND prod_price <= 6 ORDER BY prod_price;
```

## 第 6 课：用通配符进行过滤

**<font color='red'>LIKE 操作符：</font>**匹配非确定性过滤，一般都是配合「通配符」一起使用

**<font color='red'>通配符：</font>**用来匹配值的一部分的特殊字符

- 通配符搜索只能用于文本字段 (字符串)，非文本数据类型字段不能使用通配符搜索

**<font color='red'>百分号 (%) 通配符：</font>**任何字符出现任意次数 (0, 1, 多次)

**<font color='red'>下划线 (_) 通配符：</font>**匹配单个字符一次

**<font color='red'>方括号 ([]) 通配符：</font>**指定一个字符集，它必须匹配指定位置 (通配符的位置) 的一个字符 (Mysql 不支持)

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**根据不同的配置，搜索可以是区分大小写的

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**`WHERE prod_name LIKE '%'`不会匹配产品名称为 NULL 的行

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16383116458647111645864711846QFpeHP.png" alt="image-20220226163831741" style="zoom:15%;" /> **<font color=#F6C843>注意：</font>**有些 DBMS 用空格来填补字段的内容。例如某列有 50 个字符，但是只存储了 (Fish bean bag toy) 17 个字符的内容，则剩余 33 个字符用「空格」填充。如果此时需要用`F%y`来匹配 Fish bean bag toy，则会失败，因为字符后面空格，结尾并不是 y。可以修改匹配规则为`F%y%`

```mysql
-- % 通配符
-- 找出所有以词 Fish 起头的产品
SELECT prod_id, prod_name FROM Products WHERE prod_name LIKE 'Fish%';
-- 找出所有包含 bean bag 字符的产品
SELECT prod_id, prod_name FROM Products WHERE prod_name LIKE '%bean bag%';

-- _ 通配符
-- 找出包含 xx  inch teddy bear 内容的产品，一个 x 代表一个任意字符
SELECT prod_id, prod_name FROM Products WHERE prod_name LIKE '__ inch teddy bear';
```

**<font color='red'>使用通配符的技巧：</font>**

- 不要过度使用通配符。如果其他操作符能达到相同的目的，应该使用其他操作符
- 在确实需要使用通配符时，也尽量不要把它们用在搜索模式的开始处。把通配符置于开始处，搜索起来是最慢的 
- 仔细注意通配符的位置。如果放错地方，可能不会返回想要的数据

**小试牛刀：**

```mysql
-- 编写 SQL 语句，从 Products 表中检索产品名称 (prod_name) 和描述 (prod_desc)，仅返回描述中包含 toy 一词的产品
SELECT prod_name, prod_desc FROM Products WHERE prod_desc LIKE '%toy%';
-- 编写 SQL 语句，从 Products 表中检索产品名称 (prod_name) 和描述 (prod_desc)，仅返回描述中未出现 toy 一词的产品。这次，按产品名称对结果进行排序
SELECT prod_name, prod_desc FROM Products WHERE NOT prod_desc LIKE '%toy%' ORDER BY prod_name;
-- 编写 SQL 语句，从 Products 表中检索产品名称 (prod_name) 和描述 (prod_desc)，仅返回描述中同时出现 toy 和 carrots 的产品。有好几种方法可以执行此操作,但对于这个挑战题，请使用 AND 和两个 LIKE 比较
SELECT prod_name, prod_desc FROM Products WHERE prod_desc LIKE '%toy%' AND prod_desc LIKE '%carrots%';
-- 编写 SQL 语句，从 Products 表中检索产品名称 (prod_name) 和描述 (prod_desc)，仅返回在描述中以先后顺序同时出现 toy 和 carrots 的产品。提示：只需要用带有三个 % 符号的 LIKE 即可
SELECT prod_name, prod_desc FROM Products WHERE prod_desc LIKE '%toy%carrots%';
```

## 第 7 课：创建计算字段

未完待续。。。





## 数据库附录

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220404/1740441649065244O5fWiyimage-20220404174044902.png" alt="image-20220404174044902" style="zoom: 50%;" />
