Rust入门教程
============

[原文链接](http://doc.rust-lang.org/guide.html)

### 目录

1. [安装 Rust](#%E5%AE%89%E8%A3%85-rust)
2. [Hello, world!](hello-world)
3. [Cargo 介绍]()
4. [变量绑定]()
5. [条件]()
6. [函数]()
7. [注释]()
8. [复合数据类型]()
9. [匹配]()
10. [循环]()
11. [字符串]()
12. [数组、向量和切片]()
13. [标准输入]()
14. [猜谜游戏]()
15. [Crates 和模块]()
16. [测试]()
17. [指针]()
18. [模式]()
19. [方法]()
20. [闭包]()
21. [迭代器]()
22. [泛型]()
23. [特征]()
24. [任务]()
25. [宏]()
26. [不安全块]()
27. [结论]()

-------------------------
嗨！欢迎阅读本 Rust 入门教程。如果你想学习如何使用 Rust 编程，那算是来对地方了。Rust 是一个关注于“高阶-裸机”的系统级编程语言——将对机器的低阶控制与对世界的高阶抽象结合在一起（人不是计算机，高级语言的作用即充当人与机器之间的翻译）。我们认为 Rust 真的很特别，在看完本教程之后，希望你也如此。

为了展示如何开始使用 Rust，我们还是用最经典的 "Hello, World!" 做例子。然后，我们会介绍在实际开发中非常有用的工具：Cargo （程序和库皆适用）。然后，我们会谈到 Rust 的基础，用一个小程序来试验。然后再学习一些高级的特性。

听起来很有趣？那就开始吧！

### 安装 Rust

使用 Rust 的第一步当然是安装！有很多种方式可以安装 Rust，最简单的是使用`rustup`脚本。如果你使用 Linux 或 Mac，你需要做的仅仅是（注意，你不需要输入`$`符号，它标识一个命令行的开始）：

```{ignore}
$ curl -s https://static.rust-lang.org/rustup.sh | sudo sh
```

如果你使用Windows，可以下载[32-bit
installer](https://static.rust-lang.org/dist/rust-nightly-i686-w64-mingw32.exe)
或 [64-bit
installer](https://static.rust-lang.org/dist/rust-nightly-x86_64-w64-mingw32.exe)
并运行.

如果你不想要 Rust 了，也是可以的（虽然对我们来说有点沮丧）。不是每种编程语言都适合每一个人。只需要传一个参数到脚本里面：

```{ignore}
$ curl -s https://static.rust-lang.org/rustup.sh | sudo sh -s -- --uninstall
```

如果你使用的是 Windows ，只需要重新运行 `.exe` 文件，它会给你一个卸载选项。

在任何时候，你都可以重新执行这个脚本来升级 Rust （这在当前这个阶段可能会比较频繁）。Rust 还没有发布 1.0 正式版，大家讨论问题一般都是基于最新版来说的，所以你最好也升级到最新版。

有些朋友听到说安装 Rust 需要使用 `curl | sudo sh`，会有抵触情绪。因为按照上面那样安装，就得首先信任下载下来的脚本是无害的，信任 Rust 维护者不会入侵你的电脑做坏事。这是一种本能。对于有这种疑虑的人，可以 [从源码构建
Rust](https://github.com/rust-lang/rust#building-from-source) ，或者直接下载
[官方二进制包](http://www.rust-lang.org/install.html) 安装。我们承诺这种脚本安装方式不会永远持续下去：它仅仅是目前（alpha 阶段）最简单的升级方式而已。

目前，Rust 官方支持如下平台：

* Windows (7, 8, Server 2008 R2), x86 only
* Linux (2.6.18 or later, various distributions), x86 and x86-64
* OSX 10.7 (Lion) or greater, x86 and x86-64

我们在上述平台上充分测试过 Rust。另外，Android 平台也做过一些测试。

最后，补充一点关于 Windows 的说明。我们视 Windows 为第一级发布平台，但老实说，Windows 的体验没有做到像 Linux/OS X 那样好。我们在努力，如果遇到 bug，请报告给我们。每一个提交都针对 Windows 测试过的，就像其它平台一样。

好了。你已经装上 Rust 了，现在你可以打开一个终端，输入：

```{ignore}
$ rustc --version
```
你可以看到类似如下的输出：

```{ignore}
rustc 0.12.0-nightly (b7aa03a3c 2014-09-28 11:38:01 +0000)
```

看到了没？Rust 已经安装成功，恭喜！

如果没看到这个信息，你可以在下面这些地方寻求帮助：

* QQ群： 144605258；
* [IRC channel irc://irc.mozilla.org/#rust](irc://irc.mozilla.org/#rust), 可以使用 [Mibbit](http://chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust) 访问。
* [Rust 邮件列表](https://mail.mozilla.org/listinfo/rust-dev)；
* [Reddit Rust 区](http://www.reddit.com/r/rust)；
* [StackOverflow](http://stackoverflow.com/questions/tagged/rust).


### Hello World!

让我们开始第一个 Rust 程序吧！打印文字 “Hello, world!” 到屏幕上，是学习一门新语言的传统。这个的作用在于你可以确保你的编译器不只是装上了，而且工作正常。并且，打印点信息到屏幕上是再正常不过的事情了。

首先我们得建一个文件来放我们的代码。我喜欢在用户主目录下创建一个目录`projects`，来放我的所有工程文件。

本教程假设你熟悉一些基本的命令行操作（并不需要你知道很多，在语言没有正式发布之前，IDE的支持都不会太好）。Rust 也没有对编辑工具有什么特殊要求，也并不关心你的代码放在哪个地方。

好了，那我们就先创建一个工程目录。

```{bash}
$ mkdir ~/projects
$ cd ~/projects
$ mkdir hello_world
$ cd hello_world
```
如果你使用 Windows，又没有使用 PowerShell 的话，`~` 可能不起作用。自己去查询终端相关文档吧。

下面我们创建一个源文件。我会使用 `editor 文件名` 这个语法来表示编辑一个文件，但你可以使用任何你喜欢的方法做的。我们创建的文件叫 `main.rs`： 

```{bash}
$ editor main.rs
```

Rust 源文件扩展名为 `.rs` 。如果你使用超过一个单词作为你的文件名，那么就使用下划线。比如，是 `hello_world.rs`，而不是 `helloworld.rs`.

然后，打开这个文件，在里面输入：

```{rust}
fn main() {
    println!("Hello, world!");
}
```

保存，在终端里面输入：

```{bash}
$ rustc main.rs
$ ./main # or main.exe on Windows
Hello, world!
```
成功！让我们回忆一下发生了什么。

```{rust}
fn main() {

}
```

这几句定义了一个 Rust **function**，`main` 函数比较特殊：它是每个 Rust 程序的开始。第一行意思是“我声明了一个函数，名叫`main`，它没有参数，也不返回任何值”。 如果存在参数，就放在 `(` 和 `)` 中间。然后，在这个例子中，我们不准备返回什么东西，所以后面再讲关于返回值的内容。

你可能还注意到了，函数体是由花括号 `{` 和 `}` 包起来的。Rust 要求这样做，并且，建议将 `{` 放在与函数声明同一行，`{` 前面放一个空格。

接着，下面这一行：

```{rust}
    println!("Hello, world!");
```

这一行是程序主体部分。实际上有好几个地方需要注意的。第一点是这一行用4个空格缩进的，而不是tab。请配置你的编辑器，按下tab键时，插入4个空格。我们提供了一些配置示例：[各种编辑器的配置](https://github.com/rust-lang/rust/tree/master/src/etc)。

第二点是`println!()`。它是一个 Rust **宏**，这是 Rust 元编程的一个结果。如果这里是一个函数，看起来就会是：`println()`。就我们目前来看，我们没必要关心它们的不同，只需知道，当你看到`!`时，意味着你在使用一个宏，而不是一个函数。Rust 将`println!`实现成一个宏，是有一些原因的，因为涉及到一些比较高级的话题，后面再讲。

最后一件事情要注意的是： Rust 的宏与 C 语言的宏明显不同，别怕使用它。后面我们会深入了解的，现在你只要相信我们说的就行了。

接下来，`"Hello, world!"` 是一个 **字符串**，字符串在系统级编程语言中是相当复杂的话题，这里的字符串是一个 **静态分配** 的串。后面我们还会谈到不同种类的分配。我们把这个字符串作为参数传递给 `println!`，它把这个串打印到屏幕上。非常简单！

接下来看到，这一行，以分号 `;` 结尾。Rust 是一个 **面向表达式** 的语言，这意味着 Rust 中大多数东西都是表达式。而这个 `;` 用于标识当前表达式结束，新的表达式准备开始。大多数 Rust 代码都以 `;` 号结尾。后面我们还会深入了解。

最后，编译运行我们写的代码。按如下方式编译（编译器名 `rustc` 跟源代码文件名 `main.rs`）：

```{bash}
$ rustc main.rs
```

如果你有 C 或 C++ 背景，你会发现这个类似 `gcc` 或 `clang`。Rust 会生成一个二进制文件，你可以输入 `ls` 查看：

```{bash}
$ ls
main  main.rs
```
或者在 Windows 上：

```{bash}
$ dir
main.exe  main.rs
```

有两个文件：源代码文件，以 `.rs` 扩展名结尾；可执行文件 （在 Windows 中是 `main.exe`，在其它平台中是 `main`）。

```{bash}
$ ./main  # or main.exe on Windows
```
运行这个可执行文件，就打印出 `Hello, world!` 到终端里面来了。

如果你来自动态语言（如 Ruby, Python 或 JavaScript）社区，你可能不习惯编译和执行两部分开。Rust 是一个 **时间提前的编译语言**，意味着你可以编译一个程序，把它分发给其他人，他们不需要安装过 Rust。如果你给别人一个 `.rb` 或 `.py` 或 `.js` 文件，他需要安装过 Ruby/Python/Javascript（好处是一个命令就可以编译执行这个程序）。在语言设计层面，所有东西都需要权衡，Rust 做了它自己的选择。

恭喜！你已经正式写了一个 Rust 程序了。你已经变成一个 Rust 程序员咯！欢迎！

下面，我会介绍一个工具，Cargo。在真实环境中写 Rust 程序的时候，会用到它。对写小程序来说，只使用 `rustc` 就够了。但当你的工程内容增长的时候，你会希望有个东西能够帮助管理工程的方方面面，同时也能够将自己的代码方便地分享给他人。这就是 Cargo 的作用。


