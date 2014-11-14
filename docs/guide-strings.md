# Rust的字符串处理方式

字符串的处理方式对任何编程语言来说都是很重要的。如果你具有应用编程语言背景，会对系统编程语言在字符串处理上的复杂性感到吃惊，尤其是在变长字符串结构的高效的处理和动态内存申请方面将涉及很多操作系统内部的细节。幸运的是，Rust这门编程语言有很多便捷的方式帮助我们进行处理。

在Rust里，字符串是一个连续的UTF-8编码的Unicode字节流，并且字符串不包含空节点和空终止符，每个字符串使用指针进行引用。

Rust有两个字符串类型：`&str`和`String`。

## &str

&str称为`字符串切片`，字符串的字面量就是`&str`格式。

```
let string = "hello there.";
```

同其它Rust的类型一样，字符串切片也有`lifetime`，一个字符串常量就是`&'static str`。字符串切片在作为函数参数的时候，可以没有明确的`lifetime`，因为这时可以自动判断出来的：

```
fn takes_slice(slice: &str) {
    println!("Got: {}", slice);
}
```

同容器切片一样，字符串切片可以简单认为是已经存在的字符串，就像`&'static str`和`String`的实例。

`&str`主要是在栈上生成：

```
use std::str;

let x: &[u8] = &[b'a', b'b'];
let stack_str: &str = str::from_utf8(x).unwrap();
```

## String

`String`定义了具有`owned`所有权的，并且可以动态生长的字符串类型，也是UTF-8编码。`String`主要是在堆上生成。

```
let mut s = "Hello".to_string();
println!("{}", s);

s.push_str(", world.");
println!("{}", s);
```

你可以使用`as_slice()`方法将一个`String`对象强制转换为`&str`类型：

```
fn takes_slice(slice: &str) {
    printn!("Got: {}", slice);
}

fn main() {
    let s = "Hello".to_string();
    takes_slice(s.as_slice());
}
```

## 最佳实践

### String vs. &str

通常`String`用于对字符串的`owned`，`&str`用于对字符串的`borrowed`。这同`Vec<T>`与`&[T]`、`T`与`&T`的关系。所以，一般情况下使用`&str`，而不使用`String`，使用`String`会增加`lifetime`的复杂性。

### 泛型

写一个泛型的字符串，一般使用`&str`：

```
fn some_string_length(x: &str) -> unit {
    x.len()
}

fn main() {
    let s = "Hello, world";
    println!("{}", some_string_length(s)); // 12
    let s = "Hello, world".to_string();
    println!("{}", some_string_length(s.as_slice())); //12
}
```

### 比较

比较`String`与字符串常量，一般使用`as_slice()`方法进行类型转换。

```
fn compare(x: String) {
    if x.as_slice() == "Hello" {
        println!("yes");
    }
}
```

这是因为将`String`转换为`&str`类型比较容易，反之将`&str`转换为`String`涉及到堆内存分配的问题。

### 字符串的索引

你可能会认为获得字符串的某个字符成员可以这样做：

```
let s = "hello".to_string();
println!("{}", s[0]);
```

这种做法在Rust中是错误的，这是因为字符串的每个成员可能会占据不同的字节。

我们知道Unicode有3个基本的组成方式：

* 编码单元，用于存储基本数据类型
* 代码指针/unicode标量值
* 字母

一般使用`graphemes()`方法来获取`&str`的成员：

```
let s = "u͔n͈̰̎i̙̮͚̦c͚̉o̼̩̰͗d͔̆̓ͥé";
for i in s.graphemes(true) {
    println!("{}", i);
}
```

打印结果：
```
u͔
n͈̰̎
i̙̮͚̦
c͚̉
o̼̩̰͗
d͔̆̓ͥ
é
```

注意：`i`在这里是`&str`类型，一个字母由多个码点组成，所以`char`（单个字符）的概念在这里不适合。

如果你想打印每个字母单独的码点，可以使用`.char()`方法：

```
let s = "u͔n͈̰̎i̙̮͚̦c͚̉o̼̩̰͗d͔̆̓ͥé";
for i in s.chars() {
    println!("{}", i);
}
```

打印结果：

```
u
͔
n
̎
͈
̰
i
̙
̮
͚
̦
c
̉
͚
o
͗
̼
̩
̰
d
̆
̓
ͥ
͔
e
́
```

如果你想打印每个字节的码点，可以使用`.bytes()`方法：

```
let s = "u͔n͈̰̎i̙̮͚̦c͚̉o̼̩̰͗d͔̆̓ͥé";
for i in s.bytes() {
    println!("{}", i);
}
```

打印结果：

```
117
205
148
110
204
142
205
136
204
176
105
204
153
204
174
205
154
204
166
99
204
137
205
154
111
205
151
204
188
204
169
204
176
100
204
134
205
131
205
165
205
148
101
204
129
```

## 其它文档

* [the &str API documentation](http://doc.rust-lang.org/std/str/index.html)
* [the String API documentation](http://doc.rust-lang.org/std/string/index.html)

