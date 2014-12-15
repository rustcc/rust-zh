% Rust Ownership指南

本指南主要讲解Rust的Ownership，这是Rust这门编程语言唯一的、引人入胜的特性，并且开发者掌握了Ownership也将对这门编程语言有更深入的了解和掌握。Ownership使Rust实现了内存安全这个终极目标。Ownership有3个不同的概念：ownership、borrowing和lifetimes，下面，我们将对这些概念进行逐一讲解。

## 前言

在我们进行详细介绍之前，关于Ownership有2个重要的知识点：

1. Rust侧重于内存安全和运行速度，为了实现这个目标，Rust主要依托“零消耗抽象模型”，该抽象模型在运行的过程中将会消耗很少的计算机资源（内存、CPU等），Ownership是“零消耗抽象模型”最主要的实践运用。

2. 本指南分析的所有概念是在程序的编译阶段，而不关心运行阶段。

另外，Ownership的学习和掌握的曲线陡，很多初入Rust的开发者将这一特性取名为“与借用检查的斗争”，也就是说有时候开者发认为（预期）某些关于Ownership的代码是正确的，但编译器会因不满足Rust的实现规则而编译出错，这将是新手经常遇到的问题。当然，开发者的经验越来越丰富后，对这个问题的处理将会越来越顺手，也会感觉这个特性设计是如此的精妙。

## Ownership

Ownership的核心就是资源，本指南的大部分讲的都是Rust编译器对内存资源的管理，当然Ownership也包括对文件资源的管理，但这里我们重点还是关注于内存。

当你的程序申请并使用某部分内存之后，将会通过某种方式去释放这些内存，从而实现对计算机资源的有效利用。假设foo函数申请了4个字节的内存，但没有释放这部分内存，我们称这种行为为“内存泄漏”，因为每次调用foo，都会再次申请4个字节的内存，当达到一定的调用次数，将会把计算机的内存资源消耗尽。所以，需要一种方法来释放这4个字节的内存。同时对内存被释放的次数也是很重要的，必须与申请的次数相对应（不然会出现所谓的“野指针”）。

对于申请内存另一个很重要的细节是：当我们申请了一定数量的内存，如何指向这部分的内存，便于操作使用，这个指向通常通过“指针”来实现，这样就建立起了内存与操作使用内存之间的关系。

通常来看，对于操作系统语言，如：C语言，需要开发者自已操作内存申请和内存释放。下面以C语言操作堆内存为例：

```
{
    int *x = malloc(sizeof(int));
    *x = 5;
    free(x);
}
```

malloc用于申请堆内存，free用于释放堆内存。

Rust将申请内存（以及其它资源）和建立内存与操作使用之间的联系称为“Ownership”。无论什么时候操作内存，当内存“拥有句柄”——常说的“指针”，超过了使用范围，编译器将自动释放这部分内存。相应的例子如下：

```
{
    let x = box 5i;
}
```

这里的box关键字，创建了一个Box<T>实例，也就是在堆内存上申请了足够的能容纳`int`型数据的内存空间，x作为拥有句柄指向了这部分堆内存，同时Rust在编译期检测x的运行范围，并在x运行范围结束时自动释放这部分堆内存。由于编译器自动做了这些事，也就减轻了开发者时刻关注内存申请和内存释放的负担。

另外当我们将box传入一个函数作为参数时，将会发生什么呢？看下面的例子：

```
fn main() {
    let x = box 5i;
    add_one(x);
}

fn add_one(mut num: Box<int>) {
    *num += 1;
}
```

上面的代码能够正常运行，但不理想。例如，像下面这样：

```
fn main() {
    let x = box 5i;
    add_one(x);
    println!("{}", x);
}

fn add_one(mut num: Box<int>) {
    *num += 1;
}
```

将不会正常编译，报错信息如下：

```
14:15 $ rust sample.rs
sample.rs:4:20: 4:21 error: use of moved value: `x`
sample.rs:4     println!("{}", x);
                               ^
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
sample.rs:4:5: 4:23 note: expansion site
sample.rs:3:13: 3:14 note: `x` moved here because it has type `Box<int>`, which is non-copyable
sample.rs:3     add_one(x);
                        ^
error: aborting due to previous error
```

记住：内存申请和内存释放是一一对应的。在这个例子中，当我们将box传入`add_one`函数的时候，也就是调用`add_one`函数时，有了2个操作该块内存的引用：x和num，但此时Rust自动将num作为该块内存的拥有者，x本该无效的，因为x的引用转移到了num，这样来看就出现了矛盾，也就是报错的原因。

改正这个错误，我们可以在调用add_one后将ownership返回，如下：

```
fn main() {
    let x = box 5i;
    let y = add_one(x);
    println!("{}", y);
}

fn add_one(mut num: Box<int>) -> Box<int> {
    *num += 1;
    num
}
```

这里，调用`add_one`函数之后，将box这块内存返回了main函数中的y变量，因此num仅仅在add_one函数内部拥有这块内存，这种设计方式在Rust代码中相当普遍。Rust的borrowing就是临时指向别的拥有者所指向的内存，并通过&标识符标明。

## Borrowing

还是继续对前面的例子进行描述 ：

```
fn add_one(mut num: Box<int>) -> Box<int> {
    *num += 1;
    num
}
```

在这个函数中num是相应堆内存的拥有句柄，最后将拥有权返回。在现实生活中，我们可以将我的部分财产给某人一段时间，但你还是这部分财产的拥有者，你仅仅是出借给了别人，这个过程就是“出让”所有权和“借入”所有权。

Rust的ownership允许一个拥有者将所有权出借一段时间，这就是所谓的“borrowing”，下面是借入所有权的例子：

```
fn add_one(num: &mut int) {
    *num += 1;
}
```

在这个函数中，该函数从它的函数调用者借入了一个int值，再增加这个int值的值，当这个函数运行结束，num的运行范围也就结束了，这个借入过程也就结束了。

## Lifetimes

将其它引用的所有权出借给另外的引用，将是一个非常复杂的过程。假如：

1. 我有一个指向某个资源的引用。
2. 我将这个引用出借给了别人。
3. 我决定操作这块资源，并释放了这块资源，但这时别人还指向这块资源。
4. 别人准备使用这块资源。

这个时候别人将会指向不存在的资源，这就是常常讲的“野指针”。

为了弥补这个泄漏，我们要确保在第3步后第4步不会发生。Rust的ownership对这个限制的处理是通过“lifetimes”来实现的，“lifetimes”描述了引用正确使用的范围。

我们再来看看下面的例子：

```
fn add_one(num: &int) -> int {
    *num + 1
}
```

Rust有一个特性叫“lifetime省略”，它允许在某些特定的环境下可以不输入lifetime语法。对于上面这个例子，如果不采用这种省略的代码写法，可以这样：

```
fn add_one<'a>(num: &'a int) -> int {
    *num + 1
}
```

这个'a被称为`lifetime`，对于这个lifetime可以写成'a，'b，'c等多种形式，当然通常会使用更加具有描述性的名字。下面我们将深入这个语法的细节。

```
fn add_one<'a>(...)
```

这表示add_one这个函数有一个lifetime为'a，如果有两个lifetime可以像下面这样：

```
fn add_one<'a, 'b>(...)
```

在函数参数列表中，我们使用已经命名的lifetime：

```
...(num: &'a int) -> ...
```

对比&int和&'a int大部分是相同的，仅仅是lifetime的'a嵌入到了&和int之间。&int读作“对一个int型值的引用”，&'a int读作“一个指向一个int型的引用拥有lifetime为'a”。

为什么要使用lifetime呢？看下面的例子：

```
struct Foo<'a> {
    x: &'a int,
}

fn main() {
    let y = &5i;
    let f = Foo {x: y};
    printfln!("{}", f.x);
}
```

这里的struct也有了lifetime，这样使用的好处是什么呢？因为这样可以确保对Foo的引用的生存期不会超过对其内部int的引用的生存期。

## 探究运行范围

一种关联lifetime的方法是引用其可见的运行范围。如下面的例子：

```
fn main() {
    let y = &5i;    // + y goes into scope
                    // |
                    // |
                    // |
                    // + y goes out of scope
}
```

加入Foo结构体

```
struct Foo<'a> {
    x: &'a int,
}

fn main() {
    let y = &5i;            // + y goes into scope
    let f = Foo {x: y};     // + f goes into scope
                            // |
                            // |
                            // + f & y goes out of scope
}
```

f存在于y的范围内，这样才能正常运行。如果不是这样，将不会正常运行：

```
struct Foo<'a> {
    x: &'a int,
}

fn main() {
    let x;                   // + x goes into scope
                             // |
    {                        // |
        let y = &5i;         // + y goes into scope
        let f = Foo {x: y};  // + f goes into scope
        x = &f.x;            // || error here
    }
                             // |
                             // |
    println!("{}", x);       // |
}                            // + x goes out of scope
```

从这个例子可看到，f和y的运行范围小于x，但是当调用x = &f.x的时候，我们创建了x指向一个超过运行范围的值。

命名为lifetime实际上是对运行范围进行命名，给某事物一个名字也是我们深入研究它的第一步。

## 'static

命名为'static的lifetime是一个特殊形式，它表明该变量具有的lifetime为整个程序。大多数开发者会使用'static声明一个字符串：

```
let x: &'static str = "Hello, world.";
```

字符串字面量的类型为&'static str。从内存结构来看，“Hello, world.”这个字符串是存在data段中，所以x这个引用将一直存在。另一个全局变量的例子如下：

```
static FOO: int = 5i;
let x: &'static int = &FOO;
```

## 共享Ownership

前面所有讲述的例子仅假设每个资源只有一个拥有者。但实际工程当中，还有另外的情况发生，比如：1台车有4个轮胎。我们想知道一个轮胎是那台车上的，但下面的例子不正确：

```
struct Car {
    name: String,
}

struct Wheel {
    size: int,
    owner: Car,
}

fn main() {
    let car = Car { name: "Delorian".to_string() };
    for _ in range(0u, 4) {
        Wheel { size: 360, owner: car };
    }
}
```

这里我们尝试将4个轮胎与相应的车建立关联。但编译器却发现代码中的for循环存在问题，其错误如下：

```
$ rustc sample.rs 
sample.rs:13:35: 13:38 error: use of moved value: `car`
sample.rs:13         Wheel { size: 360, owner: car };
                                               ^~~
sample.rs:13:35: 13:38 note: `car` moved here because it has type `Car`, which is non-copyable
sample.rs:13         Wheel { size: 360, owner: car };
                                               ^~~
error: aborting due to previous error
```

这里我们需要1台4个轮胎指向的车，但我们不能使用Box<T>，因为它只能定义一个拥有者，所以我们使用Rc<T>这个新类型，如下所示：

```
use std::rc::Rc;

struct Car {
    name: String,
}

struct Wheel {
    size: int,
    owner: Rc<Car>,
}

fn main() {
    let car = Car { name: "Delorian".to_string() };
    let car_owner = Rc::new(car);

    for _ in range(0u, 4) {
        Wheel { size: 360, owner: car_owner.clone() };
    }
}
```

这是一个非常简单的多个拥有者的例子。与之对应，还有一个Arc<T>类型，它是在线程中的安全原子操作。

## 相关资源

http://arthurtw.github.io/2014/11/30/rust-borrow-lifetimes.html

http://rust.cc/t/yong-you-guan-xi-zhuan-yi-zu-jie-zu-jie-qi-he-sheng-cun-qi-xue-xi-bi-ji/103
