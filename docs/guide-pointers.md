% Rust指针入门指南

Rust的指针是它非常独特和迷人的功能之一。对于Rust新手来说指针也是非常困惑的主题
中的一个。它们也许也困扰着从其它支持指针的语言（例如C++）转过来的人。这个入门
指南将帮助你理解这一重要主题。

在Rust中应对所有非引用指针持有怀疑态度:处于审慎的态度使用它们，而不仅是为了讨
好编译器。每一种指针类型都有一个何时才是恰当的使用的说明。默认使用引用，除非你
是那些情况之一。

你或许对这个[速查表](#速查表)有兴趣，它给出了对各种指针的类型、名字和目的
的一个概览。

# 介绍

如果你对于指针的概念不熟悉，这儿是一个简短的介绍。指针是系统编程语言的一个基础
概念，所以理解它非常重要。

## 指针基础

当你创建了一个新的变量绑定，你也就给了一个存储在栈上特定位置的值一个名字。（如
果你不熟悉“堆”和它相对的“栈”，请看[Stack Overflow的问题](http://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap),在指南的接下来的部分
将假设你知道它们的区别。）像这样：

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

让我们来解释指针。在一些语言中，只有一种类型的‘指针’，但是在Rust中，我们有很多种。这里，我们使用最简单的一种指针，**引用(reference)**。

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
输出the个地址。

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
存储的值。这个能行：

```{rust}
let x = 5i;
let y = 8i;
let z = &y;

println!("{}", x + *z);
```

输出 `13`.

就这样！那就是指针的所有：它们指向某些你内存地址。对它们来说没有其它的了。
现在我们已经讨论了指针‘是什么’，让我们来说说“为什么”吧。

## 指针的使用

Rust的指针十分有用，但是与其它的系统语言比它有自己不同的方式。在指南接下来的
部分我们会讲Rust指针的最佳练习，不过这儿还是有一些指针在其它语言中非常有用的
使用方式：

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

那么，指针在这儿做了什么？好吧，因为指针指向一个内存地址...

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

## 指针共有的问题

我们谈到了指针，并且唱完了颂歌。那么不好的一面是什么呢？好吧，Rust试图改善所有这些问
题，但是这里有一些指针在其它语言中存在的问题：

未初始化的指针可能带来问题。譬如，这个程序会做什么？

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

`x`局限于`make_pointer`函数中，因此，一旦`make_pointer`返回，它就会失效。但是我们
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

我们说`y`是`x`的一个引用。第一个`println!`用解引用操作符`*`输出`y`所引用的值。第二
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

尽管完全安全，但一个引用在运行时的形态和一个C程序的普通指针是一样的。它带来零开销。所有的
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
函数的结尾。那么你知道这段代码将出现一个错误。我们称这段持续期叫做‘生命周期’。让我们来尝
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
编译器能判断出一直到离开作用域为止，`x`都没有被修改，因此，我们让过它，让它编过。
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

你或许会想，这种分析对于人来说是复杂的，因此对于编译器也颇有难度！这儿有一份完全讲[引用和生命周期的指南](guide-lifetimes.html),里面有生命周期里面的精彩细节，好好看看。

## Best practices

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

上述规则的必然结果局势，引用允许你宽泛的接受其它种类的指针，那是如此有用以至于你都
不用每种指针写大量变量。换言之，宁写：

```{rust}
fn succ(x: &int) -> int { *x + 1 }
```

不写

```{rust}
fn box_succ(x: Box<int>) -> int { *x + 1 }

fn rc_succ(x: std::rc::Rc<int>) -> int { *x + 1 }
```

# Boxes

`Box<T>` is Rust's 'boxed pointer' type. Boxes provide the simplest form of
heap allocation in Rust. Creating a box looks like this:

```{rust}
let x = box(std::boxed::HEAP) 5i;
```

`box` is a keyword that does 'placement new,' which we'll talk about in a bit.
`box` will be useful for creating a number of heap-allocated types, but is not
quite finished yet. In the meantime, `box`'s type defaults to
`std::boxed::HEAP`, and so you can leave it off:

```{rust}
let x = box 5i;
```

As you might assume from the `HEAP`, boxes are heap allocated. They are
deallocated automatically by Rust when they go out of scope:

```{rust}
{
    let x = box 5i;

    // stuff happens

} // x is destructed and its memory is free'd here
```

However, boxes do _not_ use reference counting or garbage collection. Boxes are
what's called an **affine type**. This means that the Rust compiler, at compile
time, determines when the box comes into and goes out of scope, and inserts the
appropriate calls there. Furthermore, boxes are a specific kind of affine type,
known as a **region**. You can read more about regions [in this paper on the
Cyclone programming
language](http://www.cs.umd.edu/projects/cyclone/papers/cyclone-regions.pdf).

You don't need to fully grok the theory of affine types or regions to grok
boxes, though. As a rough approximation, you can treat this Rust code:

```{rust}
{
    let x = box 5i;

    // stuff happens
}
```

As being similar to this C code:

```{notrust,ignore}
{
    int *x;
    x = (int *)malloc(sizeof(int));
    *x = 5;

    // stuff happens

    free(x);
}
```

Of course, this is a 10,000 foot view. It leaves out destructors, for example.
But the general idea is correct: you get the semantics of `malloc`/`free`, but
with some improvements:

1. It's impossible to allocate the incorrect amount of memory, because Rust
   figures it out from the types.
2. You cannot forget to `free` memory you've allocated, because Rust does it
   for you.
3. Rust ensures that this `free` happens at the right time, when it is truly
   not used. Use-after-free is not possible.
4. Rust enforces that no other writeable pointers alias to this heap memory,
   which means writing to an invalid pointer is not possible.

See the section on references or the [lifetimes guide](guide-lifetimes.html)
for more detail on how lifetimes work.

Using boxes and references together is very common. For example:

```{rust}
fn add_one(x: &int) -> int {
    *x + 1
}

fn main() {
    let x = box 5i;

    println!("{}", add_one(&*x));
}
```

In this case, Rust knows that `x` is being 'borrowed' by the `add_one()`
function, and since it's only reading the value, allows it.

We can borrow `x` multiple times, as long as it's not simultaneous:

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

Or as long as it's not a mutable borrow. This will error:

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

Notice we changed the signature of `add_one()` to request a mutable reference.

## Best practices

Boxes are appropriate to use in two situations: Recursive data structures,
and occasionally, when returning data.

### Recursive data structures

Sometimes, you need a recursive data structure. The simplest is known as a
'cons list':


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

This prints:

```{notrust,ignore}
Cons(1, box Cons(2, box Cons(3, box Nil)))
```

The reference to another `List` inside of the `Cons` enum variant must be a box,
because we don't know the length of the list. Because we don't know the length,
we don't know the size, and therefore, we need to heap allocate our list.

Working with recursive or other unknown-sized data structures is the primary
use-case for boxes.

### Returning data

This is important enough to have its own section entirely. The TL;DR is this:
you don't generally want to return pointers, even when you might in a language
like C or C++.

See [Returning Pointers](#returning-pointers) below for more.

# Rc and Arc

This part is coming soon.

## Best practices

This part is coming soon.

# Raw Pointers

This part is coming soon.

## Best practices

This part is coming soon.

# Returning Pointers

In many languages with pointers, you'd return a pointer from a function
so as to avoid copying a large data structure. For example:

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

The idea is that by passing around a box, you're only copying a pointer, rather
than the hundred `int`s that make up the `BigStruct`.

This is an antipattern in Rust. Instead, write this:

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

This gives you flexibility without sacrificing performance.

You may think that this gives us terrible performance: return a value and then
immediately box it up ?! Isn't that the worst of both worlds? Rust is smarter
than that. There is no copy in this code. main allocates enough room for the
`box , passes a pointer to that memory into foo as x, and then foo writes the
value straight into that pointer. This writes the return value directly into
the allocated box.

This is important enough that it bears repeating: pointers are not for
optimizing returning values from your code. Allow the caller to choose how they
want to use your output.

# Creating your own Pointers

This part is coming soon.

## Best practices

This part is coming soon.

# Patterns and `ref`

When you're trying to match something that's stored in a pointer, there may be
a situation where matching directly isn't the best option available. Let's see
how to properly handle this:

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

The `ref s` here means that `s` will be of type `&String`, rather than type
`String`.

This is important when the type you're trying to get access to has a destructor
and you don't want to move it, you just want a reference to it.

# 速查表

Here's a quick rundown of Rust's pointer types:

| Type         | Name                | Summary                                                             |
|--------------|---------------------|---------------------------------------------------------------------|
| `&T`         | Reference           | Allows one or more references to read `T`                           |
| `&mut T`     | Mutable Reference   | Allows a single reference to read and write `T`                     |
| `Box<T>`     | Box                 | Heap allocated `T` with a single owner that may read and write `T`. |
| `Rc<T>`      | "arr cee" pointer   | Heap allocated `T` with many readers                                |
| `Arc<T>`     | Arc pointer         | Same as above, but safe sharing across threads                      |
| `*const T`   | Raw pointer         | Unsafe read access to `T`                                           |
| `*mut T`     | Mutable raw pointer | Unsafe read and write access to `T`                                 |

# Related resources

* [API documentation for Box](std/boxed/index.html)
* [Lifetimes guide](guide-lifetimes.html)
* [Cyclone paper on regions](http://www.cs.umd.edu/projects/cyclone/papers/cyclone-regions.pdf), which inspired Rust's lifetime system
