**字符和字节**

字节(Byte)是计量单位，表示数据量多少，是计算机信息技术用于计量存储容量的一种计量单位，通常情况下一字节等于八位。字符(Character)计算机中使用的字母、数字、字和符号。根据编码方式的不同，一个中英文字符对应不同的个数的字节，常见的编码方式ASCII、Unicode、UTF-8（一个英文字符1个字节，一个中文3个字节）。

go中：

> 参考：https://blog.csdn.net/wohu1104/article/details/106202720

遍历字符串：索引字符串会产生其字节，而不是其字符 。在字符串上使用 for range 循环时，range 循环在每次迭代中会解码一个 UTF-8 编码 rune。

```go
func main() {
   str := "Go爱好者"
   for i:=0;i<len(str);i++{
      c := str[i]
      fmt.Printf("%d: %q [% x]\n", i, c, []byte(string(c)))
   }
   //...
   for i, c := range str {
      fmt.Printf("%d: %q [% x]\n", i, c, []byte(string(c)))
   }
   //0: 'G' [47]
   //1: 'o' [6f]
   //2: '爱' [e7 88 b1]
   //5: '好' [e5 a5 bd]
   //8: '者' [e8 80 85]
   }
```

字符串只是一堆字节字符串类型，字符串是只读的 ，其底层数据结构包括一个是指针指向字节数组的起点，另一个是字符串的长度。len() 函数判断字符串长度的时候，是判断字符的字节数而不是字符长度。 

```go
	a := "1a中国"
	fmt.Println(len(a)) //8 字符串中的字节的个数
	fmt.Println(len([]rune(a))) //4 字符的个数 统计带中文的字符串的字符个数方法之一
```

byte和rune：Go语言的两种类型的字符，一种是 byte 类型，byte 类型是 uint8 类型的一个别名 ，代表了 ASCII 码的一个字符。另一种是 rune 类型，代表一个 UTF-8 字符，当需要处理中文、日文或者其他复合字符时，则需要用到 rune 类型，rune 类型别名是 int32 类型。rune类型是为了更方便的表示类似中文的非英文字符，处理起来更加方便；而byte类型则对英文字符的处理更加友好。 



**slice底层**



**map底层**





**chan**

select





**GMP模型**



**GC**

