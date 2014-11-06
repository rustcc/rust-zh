Rust入门教程
============

[原文链接](http://doc.rust-lang.org/guide.html)

### 目录

1. [安装 Rust](#%E5%AE%89%E8%A3%85-rust)
2. [Hello, world]()
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

如果你使用的是 Windows 安装器，只需要重新执行 `.exe` 文件，它会给你一个卸载选项。

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
* [IRC channel](irc://irc.mozilla.org/#rust), 可以使用 [Mibbit](http://chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust) 访问。
* [Rust 邮件列表](https://mail.mozilla.org/listinfo/rust-dev)；
* [Reddit Rust 区](http://www.reddit.com/r/rust)；
* [StackOverflow](http://stackoverflow.com/questions/tagged/rust).

