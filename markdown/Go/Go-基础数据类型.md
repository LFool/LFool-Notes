# Go 基础数据类型

### <font color=#1FA774>Go 基本数据类型</font>

从整体来看，Go 只有三大类基本数据类型，分别为：

- `bool`类型
- 数值类型：整数、`byte`字节、`rune`类型、浮点数、复数
- 字符和`string`类型

对于`bool`类型，其实和 Java 中的`boolean`差不多，有`false`和`true`两种值，定义方式：`var b bool = true`

对于字符和`string`类型，其实也和 Java 差不多，不过值得注意的是 Go 中只有`string`关键字表示字符串，并没有为字符设置单独的关键字，这个后文还会讨论

对于数值类型，内容就比较多，分类也比较细致，下面详细介绍！

#### <font color=#9933FF>整数</font>

Go 中的整数根据位数、有无符号分为了 8 类，如下表所示：

|  类型  | 位数 | 有无符号 |                             范围                             |
| :----: | :--: | :------: | :----------------------------------------------------------: |
|  int8  |  8   |    有    |               -128 ～ 127 (-$2^7$ ～ $2^7-1$)                |
| int16  |  16  |    有    |          -32768 ～ 32767 (-$2^{15}$ ～ $2^{15}-1$)           |
| int32  |  32  |    有    |     -2147483648 ～ 2147483647 (-$2^{31}$ ～ $2^{31}-1$)      |
| int64  |  64  |    有    | -9223372036854775808 ～ 9223372036854775807 (-$2^{63}$ ～ $2^{63}-1$) |
| uint8  |  8   |    无    |                    0 ～ 255 (0 ～ $2^7$)                     |
| uint16 |  16  |    无    |                  0 ～ 65535 (0 ～ $2^{15}$)                  |
| uint32 |  32  |    无    |               0 ～ 4294967295 (0 ～ $2^{31}$)                |
| uint64 |  64  |    无    |          0 ～ 18446744073709551615 (0 ～ $2^{63}$)           |

**<font color='red'>注意：</font>**除此之外，还有一个`int`类型，它会根据计算机的位数来动态确定位数。如果是 32 位计算机，那么`int`就是 32 位有符号数；如果是 64 位计算机，那么`int`就是 64 位有符号数

#### <font color=#9933FF>byte 字节</font>

在源码中，`byte`是等价于`uint8`

```go
type byte = uint8
```

所以`byte`类型另外一大用处就是表示英文字符，也就是可以将一个纯英文字符串转化成一个`[]byte`数组，数组中的每一个元素的值即为一个英文字符的 ASCII 码值

```go
func main() {
	s := "abc"
	bytes := []byte(s)
	fmt.Println(bytes)
	fmt.Printf("%c - %c - %c\n", bytes[0], bytes[1], bytes[2])
}
// 输出
// [97 98 99]
// a - b - c
```

#### <font color=#9933FF>rune 类型</font>

如果一个字符串中有中文字符怎么办？？

中文字符使用 Unicode 编码，`uint8`类型的范围无法保存一个完整的中文，此时就有了`rune`类型，它等价于`int32`

```go
type rune = int32
```

所以`rune`类型另外一大用处就是表示中文字符，也就是可以将一个含有中文的字符串转化成一个`[]rune`数组，数组中的每一个元素即表示一个字符

```go
func main() {
	s := "abc你好"
	arr := []rune(s)
	fmt.Println(arr)
	fmt.Printf("%c - %c - %c - %c - %c\n", arr[0], arr[1], arr[2], arr[3], arr[4])
}
// 输出
// [97 98 99 20320 22909]
// a - b - c - 你 - 好
```

**<font color='red'>技巧：</font>**对于含有中文的字符串进行切片操作，先字符串转化成`[]rune`数组，然后进行切片，最后将`[]rune`数组转化回字符串即可

#### <font color=#9933FF>浮点数</font>

Go 中的浮点数只有两个，分别是`float32`和`float64`，对应 Java 中的`float`和`double`

它们俩的最大值分别是：`3.4028234663852886e+38`和`1.7976931348623157e+308`

**<font color='red'>注意：</font>**如果没有显示的给出类型，那么默认是`float64`类型

```go
var x float32 = 1.23  // float32
var y float64 = 1.23  // float64
z := 1.23             // float64
```

#### <font color=#9933FF>复数</font>

复数是有实部和虚部的数。在 Go 中，复数只有两个，分别是`complex64`和`complex128`，二者分别由`float32`和`float64`构成

同样的，如果没有显示的给出类型，那么默认是`complex128`类型

```go
var x complex64 = complex(1, 2)   // complex64
var y complex128 = complex(1, 2)  // complex128
z := complex(1, 2)                // complex128
```

**<font color='red'>OS：</font>**我感觉自己一时半会还用不上复数，以后用到了再仔细研究！！

### <font color=#1FA774>数据类型的转换</font>

在 Go 中，由于数值类型划分比较细致，所以它的类型转化也更为丰富，下面从 4 个方面进行总结

#### <font color=#9933FF>简单数值类型间转换</font>

简单数值类型的转换包括`int8`、`int16`、`int32`、`int64`、`uint8`、`uint16`、`uint32`、`uint64`、`int`、`byte`、`rune`之间的转换

它们可以使用一个简单的公式概括：`valueOfTypeB = typeB(valueOfTypeA)`，下面给出几个例子：

```go
var a int8 = 2
var b = uint8(a)  // int8 转换成 uint8

var c byte = 2
var d = int8(c)   // byte 转换成 int8
```

#### <font color=#9933FF>int 和 string 间转换</font>

如果想要将`int`类型转换成`string`类型，可以使用`strconv.Itoa()`方法；如果想要反过来转化，可以使用`strconv.Atoi()`方法

```go
var a int = 123
s := strconv.Itoa(a)    // int 转换成 string

b, err := strconv.Atoi(s) // string 转换成 int
```

**<font color='red'>注意：</font>**该转换方法只适用于对格式没有任何要求的情况

#### <font color=#9933FF>strconv.ParseXxx() 转换方法</font>

这一大类方法中一共有五种具体的方法，分别为：

```go
strconv.ParseBool(str string) (bool, error)                                  // 将 string 转换成 bool
strconv.ParseInt(s string, base int, bitSize int) (i int64, err error)       // 将 string 转换成 int64
strconv.ParseUint(s string, base int, bitSize int) (uint64, error)           // 将 string 转换成 uint64
strconv.ParseFloat(s string, bitSize int) (float64, error)                   // 将 string 转换成 float64
strconv.ParseComplex(s string, bitSize int) (complex128, error)              // 将 string 转换成 complex128
```

在上述五种方法中，有些方法中有参数`base`和`bitSize`，其中`base`表示传入字符串的进制数，`bitSize`表示传入字符串的位数

举个例子，`val, err := strconv.ParseInt("1004", 10, 8)`表示将十进制 8 位的字符串`"1004"`转换成`int64`类型，结果为 127

上面结果之所以是 127，是因为 8 位整数最大为 127，而字符串`"1004"`超过了最大值，所以取了最大值

值得注意的是，在`ParseBool()`方法中，`"1"`和`"true"`都会被转化成`true`，而`"0"`和`"false"`都会被转化成`false`，其它值都属于非法转换

#### <font color=#9933FF>strconv.FormatXxx() 转换方法</font>

这一大类方法中一共也有五种具体的方法，分别为：

```go
strconv.FormatBool(b bool) string                                            // 将 bool 转换成 string
strconv.FormatInt(i int64, base int) string                                  // 将 int64 转换成 string
strconv.FormatUint(i uint64, base int) string                                // 将 uint64 转换成 string
strconv.FormatFloat(f float64, fmt byte, prec, bitSize int) string           // 将 float64 转换成 string
strconv.FormatComplex(c complex128, fmt byte, prec, bitSize int) string      // 将 complex128 转换成 string
```

同理，在上述五种方法中，有些方法中有参数`base`和`bitSize`，其中`base`表示传入数值的进制数，`bitSize`表示传入数值的位数

举个例子，`val := strconv.FormatInt(123, 8)`表示将八进制值为`123`的的整数转换为字符串`"173"`

值得注意的是，在`FormatFloat()`方法中多了两个参数，分别为`fmt`表示转换成字符串的格式、`prec`表示转换成字符串的精度，即保留小数的位数
