# Rust的Crates和Modules

当一个项目越来越庞大、代码越来越多的时候，一个好的工程实施方式是将这些代码划分为一个部分一个部分，分而治之，并且采用private和public限定方式，定义好各部分之间的接口。为了方便这种工程实施方式，Rust中采用了模块的代码组织方式。

## 基本概念：Crates和Modules

Rust有两个不同的模块组织方式：`Crate`和`Module`。一个`Crate`相当于其它语言的`library`和`package`。Rust中采用`Cargo`这个包管理工具，来管理一个一个的`Crate`。一个`Crate`能够被生成为一个可执行文件或共享库。

每个`Crate`都有一个隐含的根模块文件，其它的模块就引用在这个根模块文件下，这其中关键的手段就是采用`Module`。

下面讲一个例子，我们将生成一个名为"phrases"的`Crate`，这个`Crate`包含了不同语言的短语，为了使例子简单，仅区分了英语和日语的"greetings"和"farewells"。该例子的模块设计如下：

**(缺少绘图，请Rust中文社区人员大力贡献)**

在这个例子中，"phrases"是这个`Crate`的名子，其它的就作为`Module`，从图中可以看出，这个`Crate`的组织方式就像一个树型结构，"phrases"是这个`Crate`的根，"english"和"chinese"两个`Module`是这个`Crate`的分枝。

下面用代码来实现这个`Crate`：

```
$ cargo new phrases  //这是在创建一个library
$ cd phrases

$ tree
.
├── Cargo.toml
└── src
    └── lib.rs

    1 directory, 2 files
```
`src/lib.rs`就是"phrases"这个`Crate`的根。

## 定义Modules

我们将使用`mod`这个关键定来定义各个模块，`src/lib.rs`里的代码如下：

```
mod english {
    mod greetings {
    }
    mod farewells {
    }
}

mod chinese {
    mod greetings {
    }
    mod farewells {
    }
}
```

上面的代码中可见，模块的定义是`mod`后紧跟模块名，模块内部代码包含在`{}`之间。在一个已知的`mod`中，可以定义下级`mod`，并通过`::`符号引用下级`mod`。在本例中的四个嵌套模块引用方式如下：`english::greetings`、`english::farewells`、`chinese::greetings`和`chinese::farewells`，这里的`greetings`和`farewells`分别分布在两个不同的模块中，所以并没有命名冲突。

由于这个`Crate`没有定义`main()`函数，所以用`Cargo`工具只能生成一个库文件：

```
$ cargo build
  Compiling phrases v0.0.1 (file:///%E6%A1%8C%E9%9D%A2/phrases)
$ ls target/
deps  examples  libphrases-066b8293cb7b2145.rlib  native
```

`libphrases-066b8293cb7b2145.rlib`就是生成后的库文件。

## 分割为多个文件组成的Crate

如果每个`Crate`仅包含一个文件，一旦工程代码增加，这个`Crate`将变得很大，而难以维护。通常都是将这个`Crate`分割成多个文件，Rust支持如下文件组织方式：

结合前面的"phrases"的描述，如果要声明一个`english`的下级模块，我们可以在`Cargo`生成的项目代码中创建`src/english.rs`或`src/english/mod.rs`代码文件，并在`src/lib.rs`文件中写入的如下代码：

```
mod english;
```

这样就可以引用`english`模块。

其项目标文件构成如下(注：在`src/english/`的下级文件中并没有写入代码)：

```
$ tree
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── english
│   │   ├── farewells.rs
│   │   ├── greetings.rs
│   │   └── mod.rs
│   ├── chinese
│   │   ├── farewells.rs
│   │   ├── greetings.rs
│   │   └── mod.rs
│   └── lib.rs
└── target
    ├── deps
        ├── examples
            ├── libphrases-066b8293cb7b2145.rlib
                └── native

                7 directories, 10 files
```

其中`src/lib.rs`就是"phrases"这个`Crate`的根，其引用下级模块的代码如下：

```
mod english;
mod chinese;
```

这样的声明，告诉Rust自动寻找`src/english.rs`、`src/chinese.rs`或者`src/english/mod.rs`、`src/chinese/mod.rs`。同样，在`english`或`chinese`这一级模块中要引用下级模块就可以分别在`src/english/mod.rs`和`src/chinese/mod.rs`进行声明：

```
mod greetings;
mod farewells;
```

同样，这个声明将告诉Rust自动寻找`src/english/greetings.rs`、`src/chinese/greetings.rs`或者`src/english/farewells/mod/rs`、`src/chinese/farewells/mod.rs`。

目前，`src/english`和`src/chinese`下面所有文件都是空的，我们将在相应的文件中定义一些函数。

在`src/english/greetings.rs`输入如下：

```
fn hello() -> String {
    "hello!".to_string()
}
```

在`src/english/farewells.rs`输入如下：

```
fn goodbye() -> String {
    "goodbye.".to_string()
}
```

在`src/japanse/greetings.rs`输入如下：

```
fn hello() -> String {
    "你好".to_string()
}
```

在`src/chinese/farewells.rs`输入如下：

```
fn goodbye() -> String {
    "再见".to_string()
}
```
下面，让我们来进行模块之间的相互调用(仅定了函数，而没有使用，在`cargo build`的时候，Rust会自动报错误)。

## 引入Crates定义的库

前面已经定义了一个名为"phrases"的`Crate`，如何使用这个库呢？

下面将创建一个`src/main.rs`文件，并输入如下代码：

```
extern crate phrases;

fn main() {
    println!("Hello in English: {}", phrases::english::greetings::hello());
    println!("Goodbye in English: {}", phrases::english::farewells::goodbye());
    println!("Hello in Chinese: {}", phrases::chinese::greetings::hello());
    println!("Goodbye in Chinese: {}", phrases::chinese::farewells::goodbye());
}
```

`extern crate`声明编译时将链接"phrases"这个`Crate`，这样才能通过`::`逐级引用"phrases"里面定义的函数。

同时，`Cargo`也认为`src/main.rs`也是一个`Crate`的根，一旦编译`src/main.rs`成功将获得一个可执行文件。目前，我们的项目包含了`src/lib.rs`和`src/main.rs`这两个`Crate`，这种组织方式对于生成可执行的`Crate`非常普遍：很多时候将函数定义在库`Crate`中，生成可执行的`Crate`调用这个库。

下面我们来编译这个项目，但会报错如下：

```
$ cargo build
   Compiling phrases v0.0.1 (file:///%E6%A1%8C%E9%9D%A2/phrases)
/phrases/src/english/greetings.rs:1:1: 3:2 warning: function is never used: `hello`, #[warn(dead_code)] on by default
/phrases/src/english/greetings.rs:1 pub fn hello() -> String {
/phrases/src/english/greetings.rs:2     "hello!".to_string()
/phrases/src/english/greetings.rs:3 }
/phrases/src/english/farewells.rs:1:1: 3:2 warning: function is never used: `goodbye`, #[warn(dead_code)] on by default
/phrases/src/english/farewells.rs:1 pub fn goodbye() -> String {
/phrases/src/english/farewells.rs:2     "goodbye".to_string()
/phrases/src/english/farewells.rs:3 }
/phrases/src/chinese/greetings.rs:1:1: 3:2 warning: function is never used: `hello`, #[warn(dead_code)] on by default
/phrases/src/chinese/greetings.rs:1 fn hello() -> String {
/phrases/src/chinese/greetings.rs:2     "你好".to_string()
/phrases/src/chinese/greetings.rs:3 }
/phrases/src/chinese/farewells.rs:1:1: 3:2 warning: function is never used: `goodbye`, #[warn(dead_code)] on by default
/phrases/src/chinese/farewells.rs:1 fn goodbye() -> String {
/phrases/src/chinese/farewells.rs:2     "再见".to_string()
/phrases/src/chinese/farewells.rs:3 }
/phrases/src/main.rs:4:38: 4:72 error: function `hello` is private
/phrases/src/main.rs:4     println!("Hello in English: {}", phrases::english::greetings::hello());
                                                                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
/phrases/src/main.rs:4:5: 4:76 note: expansion site
/phrases/src/main.rs:5:40: 5:76 error: function `goodbye` is private
/phrases/src/main.rs:5     println!("Goodbye in English: {}", phrases::english::farewells::goodbye());
                                                                  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
/phrases/src/main.rs:5:5: 5:80 note: expansion site
/phrases/src/main.rs:6:39: 6:74 error: function `hello` is private
/phrases/src/main.rs:6     println!("Hello in Chinese: {}", phrases::chinese::greetings::hello());
                                                                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
/phrases/src/main.rs:6:5: 6:78 note: expansion site
/phrases/src/main.rs:7:41: 7:78 error: function `goodbye` is private
/phrases/src/main.rs:7     println!("Goodbye in Chinese: {}", phrases::chinese::farewells::goodbye());
                                                                   ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
/phrases/src/main.rs:7:5: 7:82 note: expansion site
error: aborting due to 4 previous errors
Could not compile `phrases`.

To learn more, run the command again with --verbose.
```

通过报错的信息可见，在Rust所有的模块和函数默认情况下都是私有的。

## 定义公共接口

虽然默认情况下，Rust定义的函数、模块和变量都是私有的，但Rust提供了非常精细地定义公共接口的方法。一般使用`pub`这个关键字来定义公共接口。为了上面的例子能够正常运行，我们将在相应的模块和函数处添加`pub`关键字：

`src/lib.rs`代码修改如下：

```
pub mod english;
pub mod chinese;
```

`src/english/mod.rs`、`src/chinese/mod.rs`代码修改如下：

```
pub mod greetings;
pub mod farewells;
```

`src/english/greetings.rs`代码修改如下：

```
pub fn hello() -> String {
    "hello!".to_string()
}
```

`src/english/farewells.rs`代码修改如下：

```
pub fn goodbye() -> String {
    "goodbye.".to_string()
}
```

`src/japanse/greetings.rs`代码修改如下：

```
pub fn hello() -> String {
    "你好".to_string()
}
```

`src/chinese/farewells.rs`代码修改如下：

```
pub fn goodbye() -> String {
    "再见".to_string()
}
```

运行`cargo build`将会执行成功。

前面，在`src/main.rs`引用下级模块的函数时，输入了`phrases::english::greetings::hello()`这个太长，并重复输入了4次，不简洁。Rust提供了另一个关键字`use`来定义当前o可见泛围。

## 使用use引入Modules

Rust提供了`use`关键字能够将下级模块的可见泛围引入到当前，下面将`src/main.rs`里的代码修改如下：

```
extern crate phrases;

use phrases::english::greetings;
use phrases::english::farewells;

fn main() {
    println!("Hello in English: {}", greetings::hello());
    println!("Goodbye in English: {}", farewells::goodbye());
}
```

带有`use`关键字的两行代码，将`greetings`和`farewells`这两个模块的可见泛围提到了当前，所以，我们可以直接使用这个两个模块下面定义的函数。最佳实路就是这样，直接引入下级模块，而不是直接引入函数。

像下面这样直接引入函数：

```
extern crate phrases;

use phrases::english::greetings::hello;
use phrases::english::farewells::goodbye;

fn main() {
    println!("Hello in English: {}", hello());
    println!("Goodbye in English: {}", goodbye());
}
```

这样做在上面这个很简单的项目中，Rust不会报错，一旦项目变大，将很容易存在函数命名冲突，这里Rust将无法正常执行。比如像下面这样：

```
extern crate phrases;

use phrases::english::greetings::hello;
use phrases::chinese::greetings::hello;

fn main() {
    println!("Hello in English: {}", hello());
    println!("Goodbye in Chinese: {}", hello());
}
```

则会报错如下：

```
$ cargo build
   Compiling phrases v0.0.1 (file:///%E6%A1%8C%E9%9D%A2/phrases)
/phrases/src/main.rs:5:5: 5:40 error: a value named `hello` has already been imported in this module
/phrases/src/main.rs:5 use phrases::chinese::greetings::hello;
                                                  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
error: aborting due to previous error
Could not compile `phrases`.

To learn more, run the command again with --verbose.
```

如果我们将引入同一个模块的多个不同的下级模块，我们可以采用更加简便的代码输入方式：

```
use phrases::english::{greetings, farewells};
```

## 使用pub use在内部Module引入其它模块

我们可以在已定义`Crate`内部的其它模块中，引入函数，这样可以在既定的内部代码组织结构中添加附加接口。

下面，接合前面的例子，修改`src/main.rs`如下：

```
extern crate phrases;

use phrases::english::{greetings, farewells};
use phrases::chinese;

fn main() {
    println!("Hello in English: {}", greetings::hello());
    println!("Goodbye in English: {}", farewells::goodbye());
    println!("Hello in Chinese: {}", chinese::hello());
    println!("Goodbye in Chinese: {}", chinese::goodbye());
}
```

再修改`src/chinese/mod.rs`如下：

```
pub use self::greetings::hello;
pub use self::farewells::goodbye;
mod greetings;
mod farewells;
```

`pub use`声明，将`chinese`模块的下级模块的函数可见泛围引入到当前模块，并成为当前`chinese`模块的一个函数。

同时，要注意`pub use`或者`use`是在`mod`声明之前定义的。

编译、运行，其结果如下：

```
$ cargo build
   Compiling phrases v0.0.1 (file:///%E6%A1%8C%E9%9D%A2/phrases)

$ ./target/phrases 
Hello in English: hello!
Goodbye in English: goodbye
Hello in Chinese: 你好
Goodbye in Chinese: 再见
```
