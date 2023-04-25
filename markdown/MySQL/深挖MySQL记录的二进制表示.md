# 深挖 MySQL 记录的二进制表示

### <font color=#1FA774>记录格式介绍</font>

说来也巧，刚刚总结了一篇关于 **[InnoDB 中 row_id 的秘密](./InnoDB中row_id的秘密.html)** 的文章，介绍了 row_id 的一些细节

**InnoDB 主键生成策略：**

- 优先使用用户自定义的主键作为主键
- 如果用户没有定义主键，则选取一个不允许存储`NULL`值的 UNIQUE 键作为主键
- 如果表中连不允许存储`NULL`值的 UNIQUE 键都没有定义，则 InnoDB 会为表默认添加一个名为`row_id`的隐藏列作为主键

这里给出在 InnoDB 中一条记录的格式：

![11](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/1729521658050192kslQcB11.svg)

**变长字段长度列表：**<font color='red'>逆序</font>记录真实数据中变长字段的长度；该列的长度随着变长字段的数量和长度而变化，即无固定长度；如果无变长字段，则该列长度为 0

**NULL 值列表：**<font color='red'>逆序</font>记录真实数据中的`NULl`值字段；仅记录允许存储`NULL`的列；如果不存在允许存储`NULL`的列，那么该列就不存在；该列长度会自动补齐成整数字节 (如只有 9 个允许存储`NULL`的列，需要 9 个 bit，但是会自动补成 2 字节)

**记录头信息：**描述一些记录的属性，固定 5 字节 (40 bit)，如下图所示：

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

### <font color=#1FA774>实战分析</font>

下面建一个「无主键，无非空唯一索引，只有一个字段」的表看看：

```mysql
CREATE TABLE `table0` (
  `name` varchar(32)
) ENGINE=InnoDB;
```

然后添加三条记录：

```mysql
+--------+
| name   |
+--------+
| Hydra  |
| Trunks |
| Susan  |
+--------+
3 rows in set (0.01 sec)
```

找到该表对应的「独立表空间」数据文件`table0.ibd`，我抽出了五条记录对应的二进制数据，如下所示：

> ​      **<font color='blue'>01 00 02 00 1c</font>** 69 6e 66 69 6d 75 6d 00
>
> ​      **<font color='blue'>04 00 0b 00 00</font>** 73 75 70 72 65 6d 75 6d
>
> <font color='green'>05</font> 00 **<font color='blue'>00 00 10 00 1f</font>** <font color='orange'>00 00 00 00 02 0a</font> <font color='pink'>00 00 00 00 32 a7</font> a8 00 00 01 1c 01 10 **<font color='red'>48 79 64 72 61</font>**
>
> <font color='green'>06</font> 00 **<font color='blue'>00 00 18 00 20</font>** <font color='orange'>00 00 00 00 02 0b</font> <font color='pink'>00 00 00 00 32 a8</font> a9 00 00 01 1d 01 10 **<font color='red'>54 72 75 6e 6b 73</font>**
>
> <font color='green'>05</font> 00 **<font color='blue'>00 00 20 ff b2</font>** <font color='orange'>00 00 00 00 02 0c</font> <font color='pink'>00 00 00 00 32 ab</font> ab 00 00 01 1f 01 10 **<font color='red'>53 75 73 61 6e</font>**

前两条分别是 Infimum 记录和 Supremum 记录；后三条是我们添加的记录

为了更加清晰，我把三条用户记录标注了一下，如下图所示：

![22](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/2029561658147396gEHTGg22.svg)

首先来分析一下 next_record，我把表示 next_record 的 2 个字节单纯列出来了，并转化成十进制：

![23](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/2038291658147909SEeNm523.svg)

然后来分析一条用户真实数据：**<font color='red'>48 79 64 72 61</font>**

- 把十六进制转成十进制：72 121 100 114 97
- 十进制对应的 ASCII： H   y   d   r  a
