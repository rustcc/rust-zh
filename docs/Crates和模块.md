## Crates和模块

Rust 有一个强健的模块系统，但跟其它编程语言也有点不一样。Rust 的模块系统有两个主要的组件： **crate** 和 **模块**。

一个 crate 是 Rust 的独立编译单元。Rust 总是一次编译一个 crate，产生一个库或可执行文件。然而，可执行文件依赖于库，而一些库有依赖于其它库。为了解决这个问题，crate 可以依赖于其它 crate。

每个 crate 包含一组层级的模块。这个树以一个单一模块（叫做 **crate 根**）开始。在 crate 根上，我们可以声明其它模块，而那些模块又可以包含其它模块，随便多深都可以。

注意我们还没有提到文件相关的东西。Rust 并不施加任务特定的关系到你的文件系统和你的模块结构。这意味着，Rust 默认用一种方便的方法去文件系统上查找模块，但这个方法也是可以重载的。

够了，让我们构建点东西！先创建一个工程，叫作 `modules`：

```{bash,ignore}
$ cd ~/projects
$ cargo new modules --bin
$ cd modules
```

用编译再次核查一下我们的工作：

```{bash,notrust}
$ cargo run
   Compiling modules v0.0.1 (file:///home/you/projects/modules)
     Running `target/modules`
Hello, world!
```

非常棒！我们已经有了一个 crate 了：`src/main.rs` 就是一个 crate。那个文件里的每样东西都位于 crate 根上。一个 crate 在它的根上定义了一个 `main` 函数后，就会产生一个可执行文件。

让我们在这个 crate 里面定义一个新的模块。编辑 `src/main.rs` 看起来像下面这个样子：

```
fn main() {
    println!("Hello, world!")
}

mod hello {
    fn print_hello() {
        println!("Hello, world!")
    }
}
```

现在，我们的 crate 根上有了一个模块，名叫 `hello`。模块使用 `下划线命名法` 命名，就像函数和变量绑定那样。

在 `hello` 模块中，我们定义了一个 `print_hello` 函数，来打印我们的 hello world 信息。模块允许你切分你的程序为清晰整洁的功能单元，把共同的东西抽出来，保持不同的东西分离开。这有点像有一组货架：一所纳万物，万物得其所。

要调用函数 `print_hello`，我们使用双冒号 `::`：

```{rust,ignore}
hello::print_hello();
```

以前见过这种用法，`io::stdin()` 和 `rand::random()`，现在你可以自己做了。然而，crate 和模块有 **可见性** 相关的规则，用来精确控制哪些函数可以被外部使用。默认，模块中的每样东西都是私有的，意味着只能在同一模块中使用。下面的代码无法编译：

```{rust,ignore}
fn main() {
    hello::print_hello();
}

mod hello {
    fn print_hello() {
        println!("Hello, world!")
    }
}
```

会报下面这个错：

```{notrust,ignore}
   Compiling modules v0.0.1 (file:///home/you/projects/modules)
src/main.rs:2:5: 2:23 error: function `print_hello` is private
src/main.rs:2     hello::print_hello();
                  ^~~~~~~~~~~~~~~~~~
```

为了使它公开，使用 `pub` 关键字：

```{rust}
fn main() {
    hello::print_hello();
}

mod hello {
    pub fn print_hello() {
        println!("Hello, world!")
    }
}
```

`pub` 关键字使用的时候，有时叫作 “导出”，因为我们使这个函数可以被其它模块使用。看下面的编译运行结构：


```{notrust,ignore}
$ cargo run
   Compiling modules v0.0.1 (file:///home/you/projects/modules)
     Running `target/modules`
Hello, world!
```

太好了！对模块还可以做更多事情的，包括移动代码到单独的文件中。不过现在已经够了。
