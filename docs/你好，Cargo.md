
### 你好, Cargo！

[Cargo](http://crates.io) 是 Rust 人用来辅助管理 Rust 工程的工具。跟 Rust 一样，Cargo 目前处于 alpha 阶段，也是正在进行中的项目。然后，对很多 Rust 项目来说，它已经足够好了，所以我们从一开始就使用 Cargo 来管理我们的项目。

Cargo 管理三件事情：构建代码、下载依赖、构建依赖。最开始的时候，你的程序可能不需要什么依赖，所以我们暂时只用到第一个功能，后面会用到其它功能的。因为我们一开始就使用 Cargo，后面添加起来也非常容易。

我们用 Cargo 来重新整理一下刚才的 Hello World 程序。要做的第一件事情就是安装 Cargo。幸运的是，用前面脚本安装的 Rust 里面已经默认包含 Cargo 了。如果你是用其它方式安装的 Rust，你可以按 [check the Cargo
README](https://github.com/rust-lang/cargo#installing-cargo-from-nightlies) 这个文档中的说明进行安装。

为了 Cargo 化我们的工程，需要做两件事：创建一个 `Cargo.toml` 配置文件，然后把源代码文件放到合适的位置。

```{bash}
$ mkdir src
$ mv main.rs src/main.rs
```
Cargo 希望你的源文件放在 `src` 目录下，目的是留出上层目录放给其它文件，如 README，版权信息等与你的代码不相关的文件。Cargo 帮助我们保持工程的整洁美观。这就是：一所纳万物，万物得其所
（在一个地方放置所有东西，所有东西都在它合适的位置上）。

下一步，编辑配置文件：

```{bash}
$ editor Cargo.toml
```
请确保文件名正确：第一个字母 `C` 要大写。把下面这些放入这个文件中：

```{ignore}
[package]

name = "hello_world"
version = "0.0.1"
authors = [ "Your name <you@example.com>" ]

[[bin]]

name = "hello_world"
```

这个文件用的是 [TOML](https://github.com/toml-lang/toml) 格式. 这种格式的解释是这样的：

> TOML 目标是成为一个语义直观、易于阅读的最小化配置文件格式。它被设计为能够明确无歧义地映射到哈希表结构。TOML 很容易被各种语言解析，成为它们的数据结构。

TOML 非常像 INI 格式，但多了一些东西。

好吧，文件里面有两个 **表**：`package` 和 `bin`。第一个告诉 Cargo 关于你的包的元信息。第二个告诉 Cargo 我们要构建一个二进制文件，而不是一个库（其实我们都可以做到！）。

一旦准备好这个文件后，我们就可以开始构建了！尝试下面的指令：

```{bash}
$ cargo build
   Compiling hello_world v0.0.1 (file:///home/yourname/projects/hello_world)
$ ./target/hello_world
Hello, world!
```
哈！我们使用 `cargo build` 构建工程，用 `./target/hello_world` 来执行它。对这个例子而言，相比使用 `rustc`， 这并没有给我们带来额外的好处，但我们要为未来考虑：当我们的工程不止一个文件的时候，需要运行 `rustc` 两次，给它传一堆参数，告诉它把所有东西构建到一起。而使用 Cargo，随着我们的工程增长，我们仍然只需要敲 `cargo build` 就可以了，它会工作得很好。

你会注意到 Cargo 产生了一个新文件： `Cargo.lock`.

```{ignore,notrust}
[root]
name = "hello_world"
version = "0.0.1"
```
这个文件被 Cargo 用来确保跟踪依赖关系。目前，我们并没有什么依赖，所以显得有点奇怪。不要自己去创建这个文件，留给 Cargo 去处理。

好了！我们已经成功地使用 Carge 构建了 `hello_world`。即使我们的程序很简单，照样用了这样正确牛逼的工具，为以后的 Rust 生涯打好基础。

既然你会使用这个工具了，那么就让我们开始学习更多 Rust 语言的内容。这些基础知识会成为你 Rust 人生中强有力的武器。

