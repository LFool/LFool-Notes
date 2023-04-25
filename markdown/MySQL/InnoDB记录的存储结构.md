# InnoDB 记录的存储结构

在 MySQL 服务器中，存储引擎负责对表中数据进行读取和写入的工作，而且不同存储引擎中数据的存储结构一般不同

常见的存储引擎有 InnoDB、MyISAM 等，而 InnoDB 是 MySQL 默认的存储引擎，本片文章基于 InnoDb 存储引擎介绍记录的存储结构

### <font color=#1FA774>InnoDB 页</font>

InnoDB 将一个表中的数据存储到磁盘中，即使 MySQL 服务重启，数据也不会丢失。而真正的数据处理过程发生在内存中，所以每次都需要将磁盘中的数据加载到内存中

磁盘和内存的读写速度差了好几个数量级，所以如果每次都一条一条的将记录从磁盘加载到内存，会非常慢，而且 InnoDB 也没有这样做，它是**以页 (16KB) 作为磁盘和内存交互的基本单位**

所以一般来说，一次最少从磁盘读取 16KB 的内容到内存中，一次最少把内存中 16KB 内存刷新回磁盘中

### <font color=#1FA774>InnoDB 行格式</font>

数据库表中一行常被称之为记录，这些记录在磁盘中存储的格式就被称为之记录格式或者行格式。InnoDB 一共设计了 4 种不同类型的行格式：COMPACT、REDUNDANT、DYNAMIC、COMPRESSED

COMPACT 行格式最常使用，所以本部分只介绍 COMPACT 行格式。从整体来上说，COMPACT 行格式一共只包含两个部分：额外信息 ➕ 真实数据，如下图所示：

![11](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/1729521658050192kslQcB11.svg)

#### <font color=#9933FF>额外信息</font>

记录的额外信息中，又包含三个部分：变长字段长度列表、NULL 值列表、记录头信息

**<font color='red'>变长字段长度列表：</font>**对于一些可变长的字段，需要记录它的长度，如：`varchar(M)`。如果没有可变长的字段，那么该列表的长度就为 0，而且需要注意是**<font color='red'>逆序</font>**记录！！

关于需要用几个字节来记录一个字段的长度，判断方法如下：(W：一个字符最多需要 W 个字节表示；M：字段类型最多存储 M 个字符；L：字段实际占用字节数)

- 如果 W * M <= 255，使用 1 个字节记录字段长度
- 如果 L <= 127, 使用 1 个字节记录字段长度
- 如果 L > 127，使用 2 个字节记录字段长度

InnoDB 在读取记录的变长字段长度列表时会先查看表结构，如果字段允许存储的最大字节数 <= 255 (即：W * M <= 255)，直接认为使用 1 个字节记录字段所占字节长度

如果 InnoDB 发现 W * M > 255，但记录长度的最高位为 0，表示 L <= 127，可认为使用 1 个字节记录字段所占字节长度，否则就认为使用 2 个字节记录字段所占字节长度

之所以 W * M > 255 时可以根据最高位为否为 0 来判断使用的字节数，是因为在存储时故意用最高位来标记了一波，实际上可存储字段字节长度的只有 15bit

**<font color='red'>NULL 值列表：</font>**为了最大程度的节约内存，如果字段存储的为 NULL 值，将不会在真实数据部分记录该字段，而直接在 NULL 值列表中用 1bit 表示即可。如果没有可 NULL 的字段，那么该列表的长度为 0

首先会统计一定不为 NULL 的字段，如：主键、NOT NULL 修饰的字段，那么 NULL 值列表中就不需要存储这些字段的状态，其它字段用 1/0 来表示 NULL / NOT NULL，而且需要注意是**<font color='red'>逆序</font>**记录！！

NULL 值列表的字节数 = (可 NULL 字段个数 + 7) / 8，例：9 个可 NULL 字段，那么 NULL 值列表就为 2 字节

**<font color='red'>记录头信息：</font>**描述一些记录的属性，固定 5 字节，如下图所示：

![21](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/1945171658144717k7gofQ21.svg)

上图各属性的详细信息如下表所示：

|     名称     | 大小 (bit) |                             描述                             |
| :----------: | :--------: | :----------------------------------------------------------: |
|   预留位 1   |     1      |                           没有使用                           |
|   预留位 2   |     1      |                           没有使用                           |
| deleted_flag |     1      |                     标记该记录是否被删除                     |
| min_rec_flag |     1      |    B+ 树的每层非叶子节点中最小的目录项记录都会添加该标记     |
|   n_owned    |     4      | 一个页面中的记录会被分为若干个组，每个组中都有一个记录是「带头大哥」，其余的记录都是「小弟」<br />「带头大哥」记录的 n_owned 值代表该组中所有的记录条数<br />「小弟」记录的 n_owned 值都为 0 |
|   heap_no    |     13     |               表示当前记录在页面堆中的相对位置               |
| record_type  |     3      | 表示当前记录的类型，0 表示普通记录，1 表示 B+ 树非叶子节点的目录项记录<br />2 表示 Infimum 记录，3 表示 Supremum 记录 |
| next_record  |     16     |                   表示下一条记录的相对位置                   |

**<font color='red'>几个小细节：</font>**

- Infimum 记录和 Supremum 记录的 heap_no 最小，分别为 0 和 1。说明 Infimum 记录和 Supremum 记录在一个页面的最前面
- next_record 表示的是**相对**位置，页面中最后一条记录的 next_record 为 0，即 Supremum 记录的 next_record 为 0

#### <font color=#9933FF>真实信息</font>

首先就是三个隐藏列，其详细属性如下表所示：

|     列名     | 是否必需 | 占用空间 |          描述           |
| :----------: | :------: | :------: | :---------------------: |
|    row_id    |    否    |  6 字节  | 行 ID，唯一标识一条记录 |
|    trx_id    |    是    |  6 字节  |         事务 ID         |
| roll_pointer |    是    |  7 字节  |        回滚指针         |

**关于 row_id 字段的详细介绍可见 [InnoDB 中 row_id 的秘密](./InnoDB中row_id的秘密.html)**，后面两个字段后续文章再详细聊！！

提到 row_id 字段，就必须说一下InnoDB 主键生成策略：

- 优先使用用户自定义的主键作为主键
- 如果用户没有定义主键，则选取一个不允许为 NULL 值的 UNIQUE 键作为主键
- 如果表中没有不允许为 NULL 值的 UNIQUE 键，那么 InnoDB 会为表默认添加一个名为`row_id`的隐藏列作为主键

隐藏列后面就是该记录真实的字段数据！！下面给出一个表结构以及表中的两条记录：

```mysql
create table record_format_demo (
	c1 varchar(10),
	c2 varchar(10) not null,
	c3 varchar(10)
) charset=ascii row_format=compact; # 该表使用 ascii 字符集，一个字符是一个字节
insert into record_format_demo(c1, c2, c3) values ('aaa', 'bb', 'c'), ('eeee', 'fff', NULL)

+------+-----+------+
| c1   | c2  | c3   |
+------+-----+------+
| aaa  | bb  | c    |
| eeee | fff | NULL |
+------+-----+------+
```

表中两条记录的存储格式如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230426/03003216824492329WrVYC1.svg)

### <font color=#1FA774>溢出页</font>

一个页的大小为 16KB，如果一条记录过大导致一个页放不下怎么办？

在 COMPACT 行格式中，对于占用空间非常多的列，在记录的真实数据处只会存储该列的一部分数据，而把剩余的数据分散存储在其它几个页面中，然后在记录真实数据处用 20 字节大小存储指向这些页的地址

存储该列的一部分数据具体为 768 字节，剩余数据存储的页称之为溢出页，溢出页之间用链表相连。下图为包含一个非常长字段的记录：

<img src="https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230426/0450281682455828T2OPPr2.svg" alt="2" style="zoom:90%;" />

### <font color=#1FA774>实战分析</font>

建立一个**「无主键，无非空唯一索引，只有一个字段」**的表：(尽量最简单！！)

```mysql
CREATE TABLE `table0` (
  `name` varchar(32)
) ENGINE=InnoDB;
```

向表中插入三条数据：

```mysql
+--------+
| name   |
+--------+
| Hydra  |
| Trunks |
| Susan  |
+--------+
```

找到该表对应的「独立表空间」数据文件`table0.ibd`，抽出了五条记录对应的二进制数据，如下所示：

> ​      **<font color='blue'>01 00 02 00 1c</font>** 69 6e 66 69 6d 75 6d 00
>
> ​      **<font color='blue'>04 00 0b 00 00</font>** 73 75 70 72 65 6d 75 6d
>
> <font color='green'>05</font> 00 **<font color='blue'>00 00 10 00 1f</font>** <font color='orange'>00 00 00 00 02 0a</font> <font color='pink'>00 00 00 00 32 a7</font> a8 00 00 01 1c 01 10 **<font color='red'>48 79 64 72 61</font>**
>
> <font color='green'>06</font> 00 **<font color='blue'>00 00 18 00 20</font>** <font color='orange'>00 00 00 00 02 0b</font> <font color='pink'>00 00 00 00 32 a8</font> a9 00 00 01 1d 01 10 **<font color='red'>54 72 75 6e 6b 73</font>**
>
> <font color='green'>05</font> 00 **<font color='blue'>00 00 20 ff b2</font>** <font color='orange'>00 00 00 00 02 0c</font> <font color='pink'>00 00 00 00 32 ab</font> ab 00 00 01 1f 01 10 **<font color='red'>53 75 73 61 6e</font>**

前两条分别是 Infimum 记录和 Supremum 记录；后三条是我们添加的记录。为了更加清晰，把三条用户记录标注一下，如下图所示：

![22](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/2029561658147396gEHTGg22.svg)

首先来分析记录头信息中的 next_record，把表示 next_record 的 2 个字节 (16 bit) 单独拎出来，并转化成十进制：

![23](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/2038291658147909SEeNm523.svg)

然后再来分析一条用户真实数据：**<font color='red'>48 79 64 72 61</font>**

- 将十六进制转成十进制：72 121 100 114 97
- 十进制对应的 ASCII： H   y   d   r  a
