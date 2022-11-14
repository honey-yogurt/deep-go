# 字符与编码

## byte & rune

byte 占用1个字节，所以它和 **uint8** 没有本质区别，它可以表示 ASCII 码表中一个字符。

rune 占用4个字节，所以它和 **int32** 没有本质区别，它可以表示一个 Unicode 字符。

```go
type byte = uint8
type rune = int32
```

所以如果想要表示中文的话，只能使用 rune 类型。

# 本质

Go语⾔中的字符串以**原⽣数据类型**出现，使⽤字符串就像使⽤其他原⽣数据类型 （int、bool、float32、float64 等）⼀样。 Go 语⾔⾥的字符串的内部实现使⽤ **UTF-8** 编码。

string 的本质 其实是 byte 切片。

英⽂字⺟占⽤⼀个字节，⽽中⽂字⺟占⽤ 3 个字节，

# 实现原理

string 是 8bit 字节的集合，通常是 **UTF-8** 编码的文本。

+ string 可以为空（长度为0），但不会是 nil，空值为 "" 或者 ``;
+ string 对象不可以修改 

## 数据结构

在源码包 src/runtime/string.go 定义了 string的数据结构

```go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

+ str：字符串的首地址
+ len：字符串的长度（字节数）

在runtime包中，使用 gostringnocopy()函数来生成字符串。

```go
var str string
str = "honey"
```

字符串生成时候，会先构建 **stringStruct** 对象，再转换成 string：

```go
func gostringnocopy(str *byte) string {
  // 构建 stringStruct
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
  // 将 stringStruct 转换成 string
	s := *(*string)(unsafe.Pointer(&ss))
	return s
}
```

string 在 runtime 包中是 stringStruct 类型，对外呈现为 string 类型。

## 字符串表示

字符串使用 Unicode 编码存储字符，对于**英文字符**来说，每个 Unicode 字符编码只需要**一个字节**。

而对于非 ASCII 字符来说，其 Unicode 编码可能需要由多个字节表示，比如**中文需要三个字节**。

故，***字符串的长度实际上表现的是字节数***。



## 字符串拼接

```go
str := "honey" + "and" + "yogurt"
```

新字符串的内存空间是***一次分配***完成的，所以性能消耗主要在***拷贝数据***上。

所有待拼接的字符串都被编译器组织到一个切片中传入 concatstrings() 函数。拼接过程需要***两次遍历切片***。第一遍遍历获取总的字符串长度（字节数），据此申请内存，第二次遍历会把字符串逐个拷贝过去。

```go
// 有删减
func concatstrings(buf *tmpBuf, a []string) string {
	l := 0
	// 第一次遍历
	for i, x := range a {
		l += n
	}
	// 分配内存，返回一个 string 和切片，二者共享内存空间
	s, b := rawstringtmp(buf, l)
  // string 无法修改，只能通过切片修改
	for _, x := range a {
		copy(b, x)
		b = b[len(x):]
	}
	return s
}
```

因为 string 无法直接修改，所以 rawstringtmp() 方法**初始化了一个指定大小的 string，同时返回一个切片，二者共享同一块内存空间，后面向切片拷贝数据，也就间接修改了 string**。

```go
func rawstringtmp(buf *tmpBuf, l int) (s string, b []byte) {
	if buf != nil && l <= len(buf) {
		b = buf[:l]
		s = slicebytetostringtmp(&b[0], len(b))
	} else {
		s, b = rawstring(l)
	}
	return
}

func rawstring(size int) (s string, b []byte) {
  // 申请内存
	p := mallocgc(uintptr(size), nil, false)
  // 返回了一个 string 结构体 和 slice 结构体
	return unsafe.String((*byte)(p), size), unsafe.Slice((*byte)(p), size)
}
```



## 类型转换

### []byte 转 string

```go
string(s) // s 类型 []byte
```

这种转换需要***一次内存拷贝***。

1. 根据切片长度申请内存空间，假设内存地址为 p，长度为 len。

2. 构建 string （string.str = p, string.len = len;）。
3. 拷贝数据（将切片中数据拷贝到新申请的内存空间）

```go
// 有删减
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) string {
	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
    // 如果预留 buf 够用，则用预留 buf
		p = unsafe.Pointer(buf)
	} else {
    // 否则重新申请内存
		p = mallocgc(uintptr(n), nil, false)
	}
  // 将切片底层数组中数据拷贝到字符串 todo
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
  // 构建字符串并返回
	return unsafe.String((*byte)(p), n)
}
```

slicebytetostring() 函数会优先使用一个固定大小的 buf，当 buf 长度不够时才会申请新的内存。



### string 转 []byte

```go
[]byte(s) // s 类型为 string
```

string 转换成 []byte 也需要***一次内存拷贝***。

1. 申请切片内存空间。
2. 将 string 拷到 切片中。

```go
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
   var b []byte
   if buf != nil && len(s) <= len(buf) {
      *buf = tmpBuf{}
     // 从预留的 buf 中切出新的切片
      b = buf[:len(s)]
   } else {
     // 生成新的切片
      b = rawbyteslice(len(s))
   }
   copy(b, s)
   return b
}
```

同样使用了预留 buf，并只在该 buf 长度不够时才会申请内存。其中 rawbyteslice() 函数用于**申请新的未初始化的切片**。**由于字符串内容将完整覆盖切片的存储空间，所以可以不初始化切片从而提升分配效率**。

### 编译优化

​	当一个字符串被转换为一个字节切片时，结果切片中的底层字节序列是此字符串中存储的字节序列的一份**深复制**。 即Go运行时将为结果切片开辟一块**足够大的内存**来容纳被复制过来的所有字节。当此字符串的长度较长时，此转换开销是比较大的。 同样，当一个字节切片被转换为一个字符串时，此字节切片中的字节序列也将被**深复制**到结果字符串中。 当此字节切片的长度较长时，此转换开销同样是比较大的。 在这两种转换中，**必须使用深复制的原因是字节切片中的字节元素是可修改的，但是字符串中的字节是不可修改的**，所以一个字节切片和一个字符串是不能共享底层字节序列的（除了在创建一个 string 时）。

出于性能考虑，有时候只是应用在***临时***需要字符串的场景下，byte 切片转换成 string 时并***不会拷贝内存***，而是直接返回一个 string。这个 string 的指针 (string.str) ***指向切片的内存***。

如：

+ 使用 m[string(b)] 来**查找** map
+ 字符串拼接，如 <"+"string(b)"+">。即一个（至少有一个被衔接的字符串值为非空字符串常量的）字符串衔接表达式中的从字节切片到字符串的转换。
+ 字符串比较：string(b) == "honey"
+ 一个 for-range 循环中跟随 range 关键字的从字符串到字节切片的转换；

由于只是临时把 byte 切片转换成 string，也就**避免了因 byte 切片内容改变而导致 string 数据的变化的问题**，所以此时可以不必拷贝内存，

```go
package main

import "fmt"

func main() {
	var str = "world"
	// 这里，转换[]byte(str)将不需要一个深复制。
	for i, b := range []byte(str) {
		fmt.Println(i, ":", b)
	}

	key := []byte{'k', 'e', 'y'}
	m := map[string]string{}
	// 这个string(key)转换仍然需要深复制。
	m[string(key)] = "value"
	// 这里的转换string(key)将不需要一个深复制。
	// 即使key是一个包级变量，此优化仍然有效。
	fmt.Println(m[string(key)]) // value
}
```

注意：在最后一行中，如果在估值`string(key)`的时候有**数据竞争**的情况，则这行的输出有可能并不是`value`。 但是，无论如何，此行都不会造成恐慌（即使有数据竞争的情况发生）。

> Q：那么应该是什么呢？或者问为什么？
>
> A：

```go
package main

import "fmt"
import "testing"

var s string
var x = []byte{1023: 'x'}
var y = []byte{1023: 'y'}

func fc() {
	// 下面的四个转换都不需要深复制。
	if string(x) != string(y) {
		s = (" " + string(x) + string(y))[1:]
	}
}

func fd() {
	// 两个在比较表达式中的转换不需要深复制，
	// 但两个字符串衔接中的转换仍需要深复制。
	// 请注意此字符串衔接和fc中的衔接的差别。
	if string(x) != string(y) {
		s = string(x) + string(y)
	}
}

func main() {
	fmt.Println(testing.AllocsPerRun(1, fc)) // 1
	fmt.Println(testing.AllocsPerRun(1, fd)) // 3
}
```



### []rune 转 string

​	在一个从 rune 切片到字符串的转换中，rune 切片中的每个 rune 值将被 UTF-8 编码为一到四个字节至结果字符串中。 如果一个 rune 值是一个不合法的 Unicode 码点值，则它将被视为 Unicode 替换字符（rune）值 0xFFFD（Unicode replacement character）。 替换字符值 0xFFFD 将被UTF-8编码为三个字节 0xef 0xbf 0xbd。

### string 转 []rune

​	当一个字符串被转换为一个 rune 切片时，此字符串中存储的字节序列将被解读为一个一个 rune 的 UTF-8 编码序列。 非法的 UTF-8 编码字节序列将被转化为 Unicode 替换字符值 0xFFFD 。

​	***但是，在字符串和字节切片之间的转换中，非法的UTF-8编码字节序列将被保持原样不变***。

Go并不支持字节切片和 rune 切片之间的直接转换。我们可以用下面列出的方法来实现这样的转换：

- 利用字符串做为中间过渡。这种方法相对方便但效率较低，因为需要做两次深复制。
- 使用[unicode/utf8](https://golang.google.cn/pkg/unicode/utf8/)标准库包中的函数来实现这些转换。 这种方法效率较高，但使用起来不太方便。
- 使用[`bytes`标准库包中的`Runes`函数](https://golang.google.cn/pkg/bytes/#Runes)来将一个字节切片转换为 rune 切片。 但此包中没有将 rune 切片转换为字节切片的函数。



## 拓展与应用

### 为什么不允许修改字符串

像 c++ 语言中的 string，其本身拥有内存空间，修改 string 是支持的。但是在 go 中，string 不包含内存空间，只有一个内存指针，这样的好处是 string 变得非常轻量，可以很方便地进行传递而***不必担心内存拷贝***。

因为 string 通常指向字符串字面量，而字符串字面量存储的位置是只读段，而不是堆或者栈，所以才有了 string 不可修改的约定。

### string 和 []byte 如何取舍

string 擅长的场景：

+ 需要字符串比较场景
+ 不需要 nil 字符串的场景

[]byte 擅长的场景：

+ 修改字符串，尤其是修改粒度为1个字节的场景；
+ 函数返回值，需要 nil 表示含义的场景。



### string 类型是可比较类型

string 类型可以通过 == 、 !=、>、>=、<、<=  比较运算符来进行比较。当比较两个字符串时，它们的底层字节将逐一进行比较。如果一个字符串是另一个字符串的前缀，并且另一个字符串较长，则另一个字符串为二者之间的较大者。

>Q：比较两个字符串是否相等，不应该先比较底层的 point 是否是同一个地址吗？即会存在两个 string 对象引用同一个 ?
>
>A：TODO

上面已经提到了比较两个字符串事实上逐个比较这两个字符串中的字节。 Go编译器一般会做出如下的优化：

- 对于`==`和`!=`比较，如果这两个字符串的长度不相等，则这两个字符串肯定不相等（无需进行字节比较）。
- 如果这两个字符串底层引用着字符串切片的指针相等，则比较结果等同于比较这两个字符串的长度。

所以两个相等的字符串的比较的时间复杂度取决于它们底层引用着字符串切片的指针是否相等。 如果相等，则对它们的比较的时间复杂度为`*O*(1)`，否则时间复杂度为`*O*(n)`。

上面已经提到了，对于标准编译器，一个字符串赋值完成之后，目标字符串和源字符串将共享同一个底层字节序列。 所以比较这两个字符串的代价很小。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	bs := make([]byte, 1<<26)
	s0 := string(bs)
	s1 := string(bs)
	s2 := s1

	// s0、s1和s2是三个相等的字符串。
	// s0的底层字节序列是bs的一个深复制。
	// s1的底层字节序列也是bs的一个深复制。
	// s0和s1底层字节序列为两个不同的字节序列。
	// s2和s1共享同一个底层字节序列。

	startTime := time.Now()
	_ = s0 == s1
	duration := time.Now().Sub(startTime)
	fmt.Println("duration for (s0 == s1):", duration)

	startTime = time.Now()
	_ = s1 == s2
	duration = time.Now().Sub(startTime)
	fmt.Println("duration for (s1 == s2):", duration)
}

//duration for (s0 == s1): 10.462075ms
//duration for (s1 == s2): 136ns

```

1ms等于1000000ns！所以请尽量避免比较两个很长的不共享底层字节序列的相等的（或者几乎相等的）字符串。

### 修改字符串

前文提到，字符串不允许修改。所以想要修改一个字符串，需要从 []byte 或者 []rune 角度出发。

```go
func changeString() {
 //纯粹英⽂
 s := "zhiguogg"
 // 强制类型转换
 byteS := []byte(s)
 byteS[0] = 'c'
 fmt.Println(string(byteS))
 //纯粹中⽂
 s1 := "我不是归⼈"
 runeS := []rune(s1)
 runeS[0] = '你'
 fmt.Println(string(runeS))
}
```

如果是一个既有英文又有中文的字符串呢？

```go
 s2 := "hello,中国"
 byteS2 := []byte(s2)
 fmt.Println(byteS2)
// [104 101 108 108 111 44 228 184 173 229 155 189]

 byteS2[0] = 'l'
 byteS2[9] = 228
 byteS2[10] = 184
 byteS2[11] = 173
 //我们把第⼀个英⽂字符改成 l，把后⾯代表'国'的三个字节更改成和'中'⼀样 预测为
lello,中中
 fmt.Println(string(byteS2))
// lello,中中

 s3 := "hello,中国"
 runeS3 := []rune(s3)
 fmt.Println(runeS3)
// [104 101 108 108 111 44 20013 22269]
 runeS3[0] = 'l'
 runeS3[7] = '中'
 fmt.Println(string(runeS3))
// lello,中中
```

rune兼容了type。即Unicode字符兼容了ACSII 码。

⼀个值在从string类型向[]byte类型转换时代表着以 UTF-8 编码的字符串会被拆分成零 散、独⽴的字节。除了与 ASCII 编码兼容的那部分字符集，以 UTF-8 编码的某个单⼀ 字节是⽆法代表⼀个字符的。

其次，⼀个值在从string类型向[]rune类型转换时代表着字符串会被拆分成⼀个个 Unicode 字符。

以上任何一种方式都要**重新申请内存，然后进行内存拷贝**。即并没有修改原字符串。



不过，我们也可以效仿 runtime 的实现，同时返回一个字符串和切片，让二者共享内存，通过切片方式就可以完成**修改原字符串**。



### 常见操作

+ 调用内置函数len来获取一个字符串值的长度（此字符串中存储的字节数）。
+ 使用容器元素索引语法 aString[i] 来获取 aString 中的第 i 个字节。 表达式 aString[i] 是***不可寻址***的。换句话说，aString[i] 不可被修改。
+ 使用子切片语法 aString[start:end] 来获取 aString 的一个子字符串。 这里，start 和 end 均为 aString 中存储的字节的下标。子切片表达式和字符串共享一部分底层字节。

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var helloWorld = "hello world!"

	var hello = helloWorld[:5] // 取子字符串
	// 104是英文字符h的ASCII（和Unicode）码。
	fmt.Println(hello[0])         // 104
	fmt.Printf("%T \n", hello[0]) // uint8

	// hello[0]是不可寻址和不可修改的，所以下面
	// 两行编译不通过。
	/*
	hello[0] = 'H'         // error
	fmt.Println(&hello[0]) // error
	*/

	// 下一条语句将打印出：5 12 true
	fmt.Println(len(hello), len(helloWorld),
			strings.HasPrefix(helloWorld, hello))
}
```



注意：如果在`aString[i]`和`aString[start:end]`中，`aString`和各个下标均为常量，则编译器将在编译时刻验证这些下标的合法性，但是这样的元素访问和子切片表达式的估值结果总是非常量（这是Go语言设计之初的一个失误，但[因为兼容性的原因导致难以弥补](https://github.com/golang/go/issues/28591)）。比如下面这个程序将打引出`4 0`。

```go
package main

import "fmt"

const s = "Go101.org" // len(s) == 9

// len(s)是一个常量表达式，但len(s[:])却不是。
var a byte = 1 << len(s) / 128
var b byte = 1 << len(s[:]) / 128

func main() {
	fmt.Println(a, b) // 4 0
}
```

`a`和`b`两个变量估值不同的具体原因请阅读[移位操作类型推断规则](https://gfw.go101.org/article/operators.html#bitwise-shift-left-operand-type-deduction)和[哪些函数调用在编译时刻被估值](https://gfw.go101.org/article/summaries.html#compile-time-evaluation)。



### 使用`for-range`循环遍历字符串中的码点

`for-range`循环控制中的`range`关键字后可以跟随一个字符串，用来遍历此字符串中的码点（而非字节元素）。 字符串中非法的UTF-8编码字节序列将被解读为Unicode替换码点值`0xFFFD`。

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ囧"
	for i, rn := range s {
		fmt.Printf("%2v: 0x%x %v \n", i, rn, string(rn))
	}
	fmt.Println(len(s))
}
```

从此输出结果可以看出：

1. 下标循环变量的值并非连续。原因是下标循环变量为字符串中字节的下标，而一个码点可能需要多个字节进行UTF-8编码。
2. 第一个字符`é`由两个码点（共三字节）组成，其中一个码点需要两个字节进行UTF-8编码。
3. 第二个字符`क्षि`由四个码点（共12字节）组成，每个码点需要三个字节进行UTF-8编码。
4. 英语字符`a`由一个码点组成，此码点只需一个字节进行UTF-8编码。
5. 字符`π`由一个码点组成，此码点只需两个字节进行UTF-8编码。
6. 汉字`囧`由一个码点组成，此码点只需三个字节进行UTF-8编码。

那么如何遍历一个字符串中的字节呢？使用传统`for`循环：

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ囧"
	for i := 0; i < len(s); i++ {
		fmt.Printf("第%v个字节为0x%x\n", i, s[i])
	}
}
```

当然，我们也可以利用前面介绍的编译器优化来使用`for-range`循环遍历一个字符串中的字节元素。 对于官方标准编译器来说，此方法比刚展示的方法效率更高。

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ囧"
	// 这里，[]byte(s)不需要深复制底层字节。
	for i, b := range []byte(s) {
		fmt.Printf("The byte at index %v: 0x%x \n", i, b)
	}
}
```



### 语法糖：将字符串当作字节切片使用

我们了解到内置函数`copy`和`append`可以用来复制和添加切片元素。 事实上，做为一个特例，如果这两个函数的调用中的第一个实参为一个字节切片的话，那么第二个实参可以是一个字符串。 （对于`append`函数调用，字符串实参后必须跟随三个点`...`。） 换句话说，在此特例中，字符串可以当作字节切片来使用。

```go
package main

import "fmt"

func main() {
	hello := []byte("Hello ")
	world := "world!"

	// helloWorld := append(hello, []byte(world)...) // 正常的语法
	helloWorld := append(hello, world...)            // 语法糖
	fmt.Println(string(helloWorld))

	helloWorld2 := make([]byte, len(hello) + len(world))
	copy(helloWorld2, hello)
	// copy(helloWorld2[len(hello):], []byte(world)) // 正常的语法
	copy(helloWorld2[len(hello):], world)            // 语法糖
	fmt.Println(string(helloWorld2))
}
```

