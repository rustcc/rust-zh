% Rust指针入门指南

Rust的指针是它非常独特和迷人的功能之一。对于Rust新手来说指针也是非常困惑的主题
中的一个。它们也许也正困扰着从其它支持指针的语言（例如C++）转过来的人。这个入门
指南将帮助你理解这一重要主题。

应对Rust中所有非引用指针持有怀疑态度:处于审慎的态度使用它们，而不仅是为了讨
好编译器。每一种指针类型都有一个何时才是恰当的使用的说明。除非你是那些情况之一，
请默认使用引用。

你或许对这个[速查表](#速查表)有兴趣，它给出了对各种指针的类型、名字和目标
的一个概览。

# 介绍

如果你并不熟悉指针的概念，这儿是一个简短的介绍。指针是系统编程语言的一个非常
基础的概念，所以理解它非常重要。

## 指针基础

当你创建了一个新的变量绑定，你也就给了一个存储在栈上特定位置的值一个名字。（如
果你不熟悉“堆”和它相对的“栈”，请看[这个Stack Overflow的问题](http://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap),
在指南的接下来的部分将假设你知道它们的区别。）像这样：

```{rust}
let x = 5i;
let y = 8i;
```
|   地址   |  值   |
|----------|-------|
| 0xd3e030 | 5	   |
| 0xd3e028 | 8     |

这里，我们杜撰了一个内存地址，它们仅是一个示例值。不论如何，重点是`x`，我们为我
们的变量使用的名字，等于内存地址`0xd3e030`，在这个地址上的值是`5`。当我们查看
`x`，我们得到了相等的值，所以`x`就是`5`。

让我们来介绍指针。在一些语言中，只有一种类型的‘指针’，但是在Rust中，我们有很多种。
这里，我们将使用最简单的一种指针，Rust**引用(reference)**。

```{rust}
let x = 5i;
let y = 8i;
let z = &y;
```
|   地址  |  值      |
|-------- |----------|
|0xd3e030 | 5        |
|0xd3e028 | 8        |
|0xd3e020 | 0xd3e028 |

看出不同了吗？指针的值包含一个内存中的地址，而不是包含一个值。在这里，是`y`的
地址。`x`和`y`是`int`类型，但是`z`是`&int`类型。我们可以用格式串`{:p}`来
输出这个地址。

```{rust}
let x = 5i;
let y = 8i;
let z = &y;

println!("{:p}", z);
```

这儿将输出我们虚构的内存地址`0xd3e028`。

因为`int`和`&int`是不同的类型，便有些事情我们不能做，例如，将他们相加：

```{rust,ignore}
let x = 5i;
let y = 8i;
let z = &y;

println!("{}", x + z);
```

这儿会给我们一个错误:

```{notrust,ignore}
hello.rs:6:24: 6:25 error: mismatched types: expected `int` but found `&int` (expected int but found &-ptr)
hello.rs:6     println!("{}", x + z);
                                  ^
```

我们可以通过操作符`*`对指针解引用。解引用一个指针意味着访问这个指针中的地址所
存储的值。这是可行的：

```{rust}
let x = 5i;
let y = 8i;
let z = &y;

println!("{}", x + *z);
```

它输出 `13`。

就这样！那就是指针的全部：它们指向某些内存地址。对它们来说再没有其它的了。
现在我们已经讨论了指针‘是什么’，让我们来说说“为什么”吧。

## 指针的使用

Rust的指针十分有用，但是与其它的系统语言比它有自己不同的方式。在指南接下来的
部分我们会讲Rust指针的最佳练习，不过这儿还是有一些指针在其它语言中非常有用的
方面：

在C中，字符串是一个指向一列以空字节结尾的`char`的指针。使用字符串的唯一方式
便是对指针熟稔于心。

指针指向内存地址，这对于不是栈上的内存是非常有用的。譬如，我们的例子用到了两个
栈变量，所以我们可以给他们命名。但是如果我们在堆上分配一些内存，我们无法像栈上
一样拥有名字。在C里面，`malloc`被用于分配堆上的内存，然后返回一个指针。

作为前面两点的更普遍的变形，在任意你拥有一个大小可变的结构的时候，你需要一个指
针。你无法在编译期说清楚到底要分配多少内存，你得用一个指针来指向你要分配的内存，
然后在运行期和它打交道。

相较于按址传递的语言来说，指针对于按值传递的语言尤为有用。一般来说，语言可以做
两种抉择（下面是胡诌的语法，不是Rust的）:

```{notrust,ignore}
fn foo(x) {
    x = 5
}

fn main() {
    i = 1
    foo(i)
    // what is the value of i here?
}
```

在值传递的语言中，`foo`获得一份`i`的拷贝，所以原本的`i`不会被修改。此时，`i`仍旧是
`1`。在按址传递的的语言中，`foo`将获得一个`i`的引用，所以它的值会被改变。此时，`i`
是`5`。

那么，指针和这有什么关系呢？好吧，因为指针指向一个内存地址...

```{notrust,ignore}
fn foo(&int x) {
    *x = 5
}

fn main() {
    i = 1
    foo(&i)
    // what is the value of i here?
}
```

即使是在按值传递的语言中，在这里`i`也会是`5`。你要知道，因为参数`x`是一个指针，我们
是传递了一份拷贝给`foo`，但是因为我们赋值的那参数，它指向的是一个内存地址，所以原来
的值还是会改变。这种模式被叫做‘按值传递引用。’妙吧！

## 指针的普遍问题

我们谈到了指针，并且唱完了颂歌。那么不好的一面是什么呢？好吧，Rust试图改善所有这些问
题，但是这是指针在其它语言中存在的问题：

未初始化的指针可能带来问题。譬如，这段程序会做什么？

```{notrust,ignore}
&int x;
*x = 5; // whoops!
```

谁知道？我们只是申明了一个指针，但是没有指向任何东西，然后给它指向的内存地址设值为`5`。
但是是哪个位置呢？没人知道。这或许无害，又或是灾难。

当你同时使用指针和函数时，很容易意外的破坏指针所指向的内存。例如：

```{notrust,ignore}
fn make_pointer(): &int {
    x = 5;

    return &x;
}

fn main() {
    &int i = make_pointer();
    *i = 5; // uh oh!
}
```

`x`位于`make_pointer`函数中，因此，一旦`make_pointer`返回，它就会失效。但是我们
返回了一个指向它的内存地址，然后回到`main`中，我们试图使用这个指针，这和我们第一个
情形非常相像。设值无效的内存地址是很糟糕的。

最后一个指针大问题的例子，**别名**也能成为一个问题。两个指针指向内存中同一地址时被叫
做别名。像这样：

```{notrust,ignore}
fn mutate(&int i, int j) {
    *i = j;
}

fn main() {
  x = 5;
  y = &x;
  z = &x; //y and z are aliased


  run_in_new_thread(mutate, y, 1);
  run_in_new_thread(mutate, z, 100);

  // what is the value of x here?
}
```

在这个特意制作的例子中，`run_in_new_thread`新起一个线程，然后调用通过它的参数给定的
函数名字。因为我们有两个线程，而且它们都操作`x`的别名，我们说不清哪一个会先完成，因此，
实际上`x`的值是不确定的。更糟糕的是，如果它们中的一个把自己指向的内存搞无效了呢？
我们又有了和之前一样的问题了，我们将设值一个无效地址。

## 总结

以上是对指针作为一个一般概念的基本概述。如我们之前略有提及的，Rust拥有多种不同的指针，
而非仅是一种，并且也解决了所有我们所说到的问题。这意味着Rust的指针要比其它语言的指针
略微复杂，但这是值得的， 它没有那些简单指针有的问题。

# 引用（References)

Rust最基础的指针类型叫做‘引用’。Rust的引用看上去像这样：

```{rust}
let x = 5i;
let y = &x;

println!("{}", *y);
println!("{:p}", y);
println!("{}", y);
```

我们说`y`是一个对`x`的引用。第一个`println!`用解引用操作符`*`输出`y`所引用的值。第二
个`println!`用指针格式串输出`y`所指向的内存地址。第三个`println!`也是输出`y`所引用
的值，因为`pringln！`会自动帮我们对它进行解引用。

这儿是一个接受一个引用的函数:

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

你也可以用`&`作为一个操作符来新建一个引用，所以我们能以两种不同的方式调用这个函数：

```{rust}
fn succ(x: &int) -> int { *x + 1 }

fn main() {

    let x = 5i;
    let y = &x;

    println!("{}", succ(y));
    println!("{}", succ(&x));
}
```

这两个`println!`都将输出`6`.

当然，如果这是平常写代码，我们不必为引用费心，直接写：

```{rust}
fn succ(x: int) -> int { x + 1 }
```

引用默认是不可变的：

```{rust,ignore}
let x = 5i;
let y = &x;

*y = 5; // error: cannot assign to immutable dereference of `&`-pointer `*y`
```

它们也可以通过`mut`来创建可变的，但只有被引用者也是可变的才行。
这个能编过:

```{rust}
let mut x = 5i;
let y = &mut x;
```

这个不行:

```{rust,ignore}
let x = 5i;
let y = &mut x; // error: cannot borrow immutable local variable `x` as mutable
```

不可变指针允许有别名：

```{rust}
let x = 5i;
let y = &x;
let z = &x;
```

然而可变指针不行：

```{rust,ignore}
let mut x = 5i;
let y = &mut x;
let z = &mut x; // error: cannot borrow `x` as mutable more than once at a time
```

尽管它们完全安全，但一个引用在运行时的形态和一个C程序的普通指针是一样的。它带来零开销。所有的
安全检查由编译器在编译期做。允许这样做的理论原来叫做**局部指针（region pointers）**。
局部指针发展成我们今天所说的**生命周期（lifetimes）**。

这里有一个简单的解释：你希望这段代码编译通过吗？

```{rust,ignore}
fn main() {
    println!("{}", x);
    let x = 5;
}
```

大概不想。因为你知道在`x`声明之处起到离开所在作用域为止，它是合法的。在这里，就是`main`
函数的结尾。那么你知道这段代码将引起一个错误。我们称这段持续期叫做‘生命周期’。让我们来尝
试一个更复杂的例子：

```{rust}
fn main() {
    let x = &mut 5i;

    if *x < 10 {
        let y = &x;

        println!("Oh no: {}", y);
        return;
    }

    *x -= 1;

    println!("Oh no: {}", x);
}
```

这儿，我们在`if`里面我们从`x`借用了一个指针（译者注：注意这个指针是可变的）。然而
编译器能判断出一直到离开作用域为止，`x`都没有被修改，因此，我们放过它，让它编过。
这个就不行了：

```{rust,ignore}
fn main() {
    let x = &mut 5i;

    if *x < 10 {
        let y = &x;
        *x -= 1;

        println!("Oh no: {}", y);
        return;
    }

    *x -= 1;

    println!("Oh no: {}", x);
}
```

编译器给我们这个错误:

```{notrust,ignore}
test.rs:5:8: 5:10 error: cannot assign to `*x` because it is borrowed
test.rs:5         *x -= 1;
                  ^~
test.rs:4:16: 4:18 note: borrow of `*x` occurs here
test.rs:4         let y = &x;
                          ^~
```

你或许会想，这种分析对于人来说是复杂的，因此对于编译器也颇有难度！这儿有一份完全讲
[引用和生命周期的指南](guide-lifetimes.html),里面有生命周期里面的精彩细节，好好
看看。

## 最佳实践

一般来说用栈分配优先于堆分配。只要允许的时候，优先使用引用来指向栈分配的信息。因此，
引用是你默认应该使用的类型，除非你有明确的原因来使用其它类型。涉及何时才是正确使用
其它几种类型的指针的话题，在它们各自的最佳实践章节中。

当你要用一个指针又不想持有产权（ownership）的时候，那就使用引用。引用仅仅借用产权，
当你不需要产权的时候，这样做起来更显优雅。换句话说，宁写：

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

不写

```{rust}
fn succ(x: Box<int>) -> int { *x + 1 }
```

上述规则的必然结果就是，引用允许你宽泛的接受其它种类的指针，那是有用的，因此你都
不用为每种指针写大量变量。换言之，宁写：

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

不写

```{rust}
fn box_succ(x: Box<int>) -> int { *x + 1 }

fn rc_succ(x: std::rc::Rc<int>) -> int { *x + 1 }
```

# Boxes

`Box<T>`是Rust的‘boxed指针’类型。boxes提供了在Rust中最简单的堆分配形式。像这样来
创建一个box：

```{rust}
let x = box(std::boxed::HEAP) 5i;
```

`box`是实施'placement new'的一个关键字，我们过会儿就会讲到。`box`对于创建各种堆分配
类型非常有用，但是现在还没有完全完成。与此同时，`box`的类型默认是`std::boxed::HEAP`，
所以你可以省略它：

```{rust}
let x = box 5i;
```

正如你可能设想是从`HEAP`的，boxes就是堆分配的。它们将在离开作用域的时候被Rust自动释放。

```{rust}
{
    let x = box 5i;

    // stuff happens

} // x is destructed and its memory is free'd here
```

然而，boxes不使用引用计数和垃圾回收。boxes被叫做**affine type(仿射类型？)**。这意味着
Rust编译器在编译期决定box什么时候开始和结束作用域，然后在那儿插入正确的调用。此外，box
是一种特殊的affine type，被称为**区域（region）**。你可以通过[Cyclone编程语言的这篇论](http://www.cs.umd.edu/projects/cyclone/papers/cyclone-regions.pdf)
文进一步了解区域。

即使要理解boxes，但你无需透彻理解affine type和区域的理论。做一个粗糙的类比，你可以把这
样的Rust代码：

```{rust}
{
    let x = box 5i;

    // stuff happens
}
```

当作是类似这样的C代码：

```{notrust,ignore}
{
    int *x;
    x = (int *)malloc(sizeof(int));
    *x = 5;

    // stuff happens

    free(x);
}
```

当然，这只是一个10,000英尺高空的俯瞰。例如，它去掉了析构函数。但是大体想法是对的：你抓住
了`malloc/free`的语义，并做一些提升：

1. 它不可能错误的分配内存大小了，因为Rust根据类型算出它。
2. 你不会忘记`free`你分配的内存了，因为Rust为你做了。
3. Rust确保这个`free`发送在正确的时机——当它真正没有被用到了。释放后使用是不可能的。
4. Rust强制这块内存没有其它可写指针别名，意味着写一个非法指针是不可能的。

进一步了解生命周期工作的细节请看引用一节或[生命周期指南](guide-lifetimes.html)。

一起使用boxes和引用是很常见的。例如：

```{rust}
fn add_one(x: &int) -> int {
    *x + 1
}

fn main() {
    let x = box 5i;

    println!("{}", add_one(&*x));
}
```

在这个例子中，Rust知道`x`被`add_one()`函数借用了，并且因为它只是读了值，所有是
允许的。

只要不是同时进行，我们可以借用多次`x`：

```{rust}
fn add_one(x: &int) -> int {
    *x + 1
}

fn main() {
    let x = box 5i;

    println!("{}", add_one(&*x));
    println!("{}", add_one(&*x));
    println!("{}", add_one(&*x));
}
```

或者只要不是可变借用。这个会报错：

```{rust,ignore}
fn add_one(x: &mut int) -> int {
    *x + 1
}

fn main() {
    let x = box 5i;

    println!("{}", add_one(&*x)); // error: cannot borrow immutable dereference 
                                  // of `&`-pointer as mutable
}
```

注意我们把`add_one()`的签名变成了要求一个可变引用。

## Best practices

box在两种情况下的应用是恰当的：递归的数据结构，和偶尔会有的，返回数据。

### 递归数据结构

有时，你需要一个递归的数据结构。最简单的被叫做'cons list':

```{rust}
#[deriving(Show)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}

fn main() {
    let list: List<int> = Cons(1, box Cons(2, box Cons(3, box Nil)));
    println!("{}", list);
}
```

这个输出:

```{notrust,ignore}
Cons(1, box Cons(2, box Cons(3, box Nil)))
```

在`Cons`枚举变量中引用另外一个`List`的必须是一个box，因为我们不知道这个链表的长度。
因为我们不知道这个链表的长度，我们就不知道大小，因此，我们需要堆分配我们的链表。

配合递归或者其它未知大小的数据结构一起工作是boxes的主要用况。

### 返回数据

这个主题已经完全重要到可以拥有用它自己的章节了。太长，不想读的话，就是这个意思咯：
你基本不想返回指针，即使你可能是用像C或C++这种语言。

更多细节看下面的[返回指针](#返回指针)。

# Rc and Arc

This part is coming soon.

## Best practices

This part is coming soon.

# Raw Pointers

This part is coming soon.

## Best practices

This part is coming soon.

# Returning Pointers

在许多有指针的语言中，你从函数返回一个指针以避免拷贝一个大的数据结构。例如：

```{rust}
struct BigStruct {
    one: int,
    two: int,
    // etc
    one_hundred: int,
}

fn foo(x: Box<BigStruct>) -> Box<BigStruct> {
    return box *x;
}

fn main() {
    let x = box BigStruct {
        one: 1,
        two: 2,
        one_hundred: 100,
    };

    let y = foo(x);
}
```

这个想法是依赖传递一个box，你只拷贝要给指针而不是由一百个`int`组成的`BigStruct`。

这在Rust中是一个错误的模式。相反的，应这样写：

```{rust}
struct BigStruct {
    one: int,
    two: int,
    // etc
    one_hundred: int,
}

fn foo(x: Box<BigStruct>) -> BigStruct {
    return *x;
}

fn main() {
    let x = box BigStruct {
        one: 1,
        two: 2,
        one_hundred: 100,
    };

    let y = box foo(x);
}
```

这在不牺牲性能的情况下给了你灵活性。

你或许想这给我们糟糕的性能：返回一个值然后立马对它装箱?!这不是世界上最烂的吗？Rust比
那要更智能。在这份代码里面不会有拷贝。主函数为box分配了足够的空间，传递一个指向那块内存
的指针进入foo当作x，然后foo直接往那个指针里面写值。这把返回值直接写入了分配好的box。[^注1]

这是非常重要值得反复重申的一点：指针不是为了优化你代码中的返回值。允许调用者选择按他们所
希望的来使用你的输出。

# Creating your own Pointers

This part is coming soon.

## Best practices

This part is coming soon.

# 模式和`ref`

当你试图匹配某些存储在指针中的东西，可能存在直接匹配并不是最佳选择的情况。让我们看如何
来恰当的处理这个例子：

```{rust,ignore}
fn possibly_print(x: &Option<String>) {
    match *x {
        // BAD: cannot move out of a `&`
        Some(s) => println!("{}", s)

        // GOOD: instead take a reference into the memory of the `Option`
        Some(ref s) => println!("{}", *s),
        None => {}
    }
}
```

`ref s`在这儿意味着`s`将是类型`&String`，而不是类型`String`。

这在你试图访问一个拥有析构函数的类型并且只想引用而不想移动它的时候是非常重要的。

# 速查表

这儿是一个Rust指针类型的快速概览：

| 类型         | 名字                | 概要                                                             |
|--------------|---------------------|-----------------------------------------------------------------|
| `&T`         | 引用                | 允许一个或多个引用读取 `T`                                        |
| `&mut T`     | 可变引用            | 允许仅一个引用读写`T`                                             |
| `Box<T>`     | Box                 | 堆分配一个唯一持有者可读写的`T`                                   |
| `Rc<T>`      | "arr cee" 指针      | 堆分配一个可多人读取的`T`                                         |
| `Arc<T>`     | Arc 指针            | 同上，但是线程间共享是安全的                                      |
| `*const T`   | Raw 指针            | 不安全读访问`T`                                                  |
| `*mut T`     | 可变 raw 指针       | 不安全读写访问`T`                                                |

# 相关资料

* [Box的API文档](std/boxed/index.html)
* [生命周期指南](guide-lifetimes.html)
* [Cyclone关于区域的论文](http://www.cs.umd.edu/projects/cyclone/papers/cyclone-regions.pdf), 正是它启发了Rust的生命周期系统。

[^注1]: 这段话的描述看上去是不明确的，只说写入那个指针，没有明确说那个指是不是y，或说y是不是以隐式参数传入。