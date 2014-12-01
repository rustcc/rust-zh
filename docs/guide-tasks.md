% Rust 任务和通信指南

# 简介（Introduction）

Rust 通过很多核心库原语提供安全的并发抽象。这个指南会描述 Rust 的并发模型，以及它与 Rust 的类型系统是如何相联系的，并且引入创建并发程序的基本库抽象。

任务提供失败隔离和恢复。当 Rsut 程序中由于显式调用 `panic!()` 而出现致命错误，断言失败，或者另一个无效的操作，运行时系统销毁整个任务。不像 Java 和 C++ 语言中那样，没有办法能够 `catch` 异常。取而代之的是任务可以监控彼此来查看是否 panic。

任务使用 Rust 的类型系统提供强健的内存安全保障。特别地，类型系统保证任务不能从共享可变状态诱导数据竞争。

# 基础（Basics）

简单讲，创建一个任务就是用闭包为参数调用 `spawn` 函数。`spawn` 在新任务中执行闭包。

```{rust}
# use std::task::spawn;

// Print something profound in a different task using a named function
fn print_message() { println!("I am running in a different task!"); }
spawn(print_message);

// Alternatively, use a `proc` expression instead of a named function.
// The `proc` expression evaluates to an (unnamed) proc.
// That proc will call `println!(...)` when the spawned task runs.
spawn(proc() println!("I am also running in a different task!") );
```

Rust 中任务不是语言语义中的概念，而是 Rust 的类型系统提供所有必要的工具来实现安全并发：特别是所有权（ownership）。语言将实现细节留给标准库。

`spawn` 函数类型签名非常简单：`fn spawn(f: proc())`。因为它只接收过程（proc）作为参数，而过程只包含拥有的数据，`spawn` 可以安全地将整个过程和所有相关状态移交给一个完全不同的任务来执行。像所有的闭包一样，传递给 `spawn` 的函数可以捕获任务之间的环境。

```{rust}
# use std::task::spawn;
# fn generate_task_number() -> int { 0 }
// Generate some state locally
let child_task_number = generate_task_number();

spawn(proc() {
    // Capture it in the remote task
    println!("I am child number {}", child_task_number);
});
```

## 通信（Communication）

现在我们已经产生一个新任务了，如果能够与其通信就很 Nice 了。对于这个，我们使用*信道(channels)*。信道仅仅是一对终端：一个用于发送消息，另一个用来接收消息。

创建信道最简单的方式是使用 `channel` 函数创建一个 `(Sender, Receiver)` 对儿。用 Rust 的说法就是**发送者(sender)**是信道的发送终端，**接收者(receiver)**是接收终端。考虑下面并发计算两个结果的例子：

```{rust}
# use std::task::spawn;

let (tx, rx): (Sender<int>, Receiver<int>) = channel();

spawn(proc() {
    let result = some_expensive_computation();
    tx.send(result);
});

some_other_expensive_computation();
let result = rx.recv();
# fn some_expensive_computation() -> int { 42 }
# fn some_other_expensive_computation() {}
```

让我们详细检查这个例子。首先，`let` 陈述创建一个用于发送和接收整数的流（`let` 左边 `(tx, rx)` 是析构 let 的一个示例：模式将元组按组成分开）。

```{rust}
let (tx, rx): (Sender<int>, Receiver<int>) = channel();
```

子任务将使用发送端发送数据给父任务，其父任务等待接收接收端上的数据。接下来的陈述 spawn 子任务。

```{rust}
# use std::task::spawn;
# fn some_expensive_computation() -> int { 42 }
# let (tx, rx) = channel();
spawn(proc() {
    let result = some_expensive_computation();
    tx.send(result);
});
```

注意任务闭包的创建隐式地将 `tx` 转移到子任务：闭包在其环境中捕获 `tx`。`Sender` 和 `Receiver` 都是可以发送的类型而且可以被任务捕获或在任务之间转移。在示例中，子任务运行一个大开销（expensive）的计算，然后通过捕获到的信道发送结果。

最后，父任务继续执行其它的大开销计算，然后等待子任务的结果到达接收端：

```{rust}
# fn some_other_expensive_computation() {}
# let (tx, rx) = channel::<int>();
# tx.send(0);
some_other_expensive_computation();
let result = rx.recv();
```

`channel` 创建的 `Sender` 和 `Receiver` 对儿允许单个发送端和单个接收端高效通信，但多个发送端不能使用单个 `Sender` 值，多个接收端不能使用单个 `Receiver` 值。要是我们的示例需要从许多任务计算多个结果呢？下面的程序的类型有问题：

```{rust,ignore}
# fn some_expensive_computation() -> int { 42 }
let (tx, rx) = channel();

spawn(proc() {
    tx.send(some_expensive_computation());
});

// ERROR! The previous spawn statement already owns the sender,
// so the compiler will not allow it to be captured again
spawn(proc() {
    tx.send(some_expensive_computation());
});
```

而我们需要克隆 `tx`，它允许多个发送端。

```{rust}
let (tx, rx) = channel();

for init_val in range(0u, 3) {
    // Create a new channel handle to distribute to the child task
    let child_tx = tx.clone();
    spawn(proc() {
        child_tx.send(some_expensive_computation(init_val));
    });
}

let result = rx.recv() + rx.recv() + rx.recv();
# fn some_expensive_computation(_i: uint) -> int { 42 }
```

克隆 `Sender` 产生一个相同信道的句柄，允许多个任务发送数据到单个接收端。它在内部将信道升级以支持这个功能，意思是没有克隆的信道可以避免需要处理多个发送端的开销。但是这一点不影响信道的使用：升级是透明的。

注意上面克隆的示例是设法巧妙做到的，因为你也可以简单地用三个 `Sender` 对，但它只是用来阐明这点的。作为参考，用多个流应该是写成下面示例这样：

```{rust}
# use std::task::spawn;

// Create a vector of ports, one for each child task
let rxs = Vec::from_fn(3, |init_val| {
    let (tx, rx) = channel();
    spawn(proc() {
        tx.send(some_expensive_computation(init_val));
    });
    rx
});

// Wait on each port, accumulating the results
let result = rxs.iter().fold(0, |accum, rx| accum + rx.recv() );
# fn some_expensive_computation(_i: uint) -> int { 42 }
```

## 后台计算：未来（Background computations: Futures）

有了 `sync::Future`， Rust 有一种机制用来请求计算，之后再获取结果。

下面基本的示例阐明了这点：

```{rust}
use std::sync::Future;

# fn main() {
# fn make_a_sandwich() {};
fn fib(n: u64) -> u64 {
    // lengthy computation returning an uint
    12586269025
}

let mut delayed_fib = Future::spawn(proc() fib(50));
make_a_sandwich();
println!("fib(50) = {}", delayed_fib.get())
# }
```

调用 `future::spawn` 立即返回一个 `future` 对象而不管运行 `fib(50)` 需要多久。然后你就可以在 `fib` 计算正在运行时悠闲地吃一块三明治了。方法执行的结果可以通过在未来对象上调用 `get` 得到。这个调用会阻塞直到结果出来（*即*计算完成）。注意未来需要是可变的以便在下次调用 `get` 时保存结果。

下面是另一个例子，演示未来是怎样允许你在后台计算的。负载会分布到可用的（CPU）核心上。

```{rust}
# use std::num::Float;
# use std::sync::Future;
fn partial_sum(start: uint) -> f64 {
    let mut local_sum = 0f64;
    for num in range(start*100000, (start+1)*100000) {
        local_sum += (num as f64 + 1.0).powf(-2.0);
    }
    local_sum
}

fn main() {
    let mut futures = Vec::from_fn(200, |ind| Future::spawn( proc() { partial_sum(ind) }));

    let mut final_res = 0f64;
    for ft in futures.iter_mut()  {
        final_res += ft.get();
    }
    println!("π^2/6 is not far from : {}", final_res);
}
```

## 无拷贝共享：Arc

为了在任务间共享数据，第一种途径会是仅仅使用我们之前见过的信道。这样每个任务都会拷贝一份共享数据。在某些情况下，这会显著增加内存浪费而且需要拷贝比必需的数据更多的相同数据。

为了解决这个问题，可以使用 Rust 的 `sync` 库实现的原子引用计数（Atomically Reference Counted）封装（`Arc`）。有了 Arc，数据就不再需要拷贝给每一个任务了。Arc 作为共享数据的引用并且只有这个引用是被共享和克隆的。

下面是一个小的示例演示如何使用 Arc。我们希望在单个大的浮点数向量上并发地运行多个计算。每个任务需要整个向量来履行职责。

```{rust}
use std::num::Float;
use std::rand;
use std::sync::Arc;

fn pnorm(nums: &[f64], p: uint) -> f64 {
    nums.iter().fold(0.0, |a, b| a + b.powf(p as f64)).powf(1.0 / (p as f64))
}

fn main() {
    let numbers = Vec::from_fn(1000000, |_| rand::random::<f64>());
    let numbers_arc = Arc::new(numbers);

    for num in range(1u, 10) {
        let task_numbers = numbers_arc.clone();

        spawn(proc() {
            println!("{}-norm = {}", num, pnorm(task_numbers.as_slice(), num));
        });
    }
}
```

`pnorm` 函数在向量上执行简单的计算（计算给定幂次的项的总和然后取这个值的反幂次）。向量上的 Arc 是通过下面创建的：

```{rust}
# use std::rand;
# use std::sync::Arc;
# fn main() {
# let numbers = Vec::from_fn(1000000, |_| rand::random::<f64>());
let numbers_arc = Arc::new(numbers);
# }
```

克隆是通过过程（procedure）被每个任务捕捉到的。这只会拷贝封装而不是它的内容。在任务的过程内部，被捕捉到的 Arc 引用可以作为基本的（underlying）向量的共享引用或者像是局部的。

```{rust}
# use std::rand;
# use std::sync::Arc;
# fn pnorm(nums: &[f64], p: uint) -> f64 { 4.0 }
# fn main() {
# let numbers=Vec::from_fn(1000000, |_| rand::random::<f64>());
# let numbers_arc = Arc::new(numbers);
# let num = 4;
let task_numbers = numbers_arc.clone();
spawn(proc() {
    // Capture task_numbers and use it as if it was the underlying vector
    println!("{}-norm = {}", num, pnorm(task_numbers.as_slice(), num));
});
# }
```

# 处理任务 panic（Handling task panics）

Rust 有一个内置机制用来抛出异常。`panic!()` 宏（也可以用错误字符串作为参数：`panic!(~reason)`）和 `assert!` 结构体（高效地调用 `panic!()` 如果布尔表达式为假）都是抛出异常的方法。当任务抛出异常时，任务解开其堆栈---运行析构器并一路释放内存---然后退出。与 C++ 中的异常不一样，Rust 中的异常在单个任务中是不可恢复的：一旦任务 panic，没有捕获异常的方法。

尽管任务不能从 panic 中恢复，但任务可以通知彼此的 panic。处理 panic 任务最简单的方法是用 `try` 函数，这与 `spawn` 相似，但会立即阻塞，等待子任务完成。`try` 返回 `Result<T, Box<Any + Send>>` 类型的值。`Result` 是有两个变量的 `enum` 类型：`Ok` 和 `Err`。这种情况，因为 `Result` 的类型参数是 `int` 和 `()`，所以调用者可以在结果上进行模式匹配检查是带有 `int` 域（代表成功的结果）的 `Ok` 结果还是 `Err` 结果（代表终止错误）

```{rust}
# use std::task;
# fn some_condition() -> bool { false }
# fn calculate_result() -> int { 0 }
let result: Result<int, Box<std::any::Any + Send>> = task::try(proc() {
    if some_condition() {
        calculate_result()
    } else {
        panic!("oops!");
    }
});
assert!(result.is_err());
```

与 `spawn` 不同，用 `try` 产生的函数可以返回一个值，这个值会在 [`Result`] 枚举中由 `try` 负责任地传回给调用者。如果子任务成功地终止了，`try` 会返回一个 `Ok` 结果；如果子任务 panic 了，`try` 会返回一个 `Error` 结果。

[`Result`]: std/result/index.html

> *注意:* 当前 panic 的任务不会产生有用的错误值（`try` 总是返回 `Err(())`）。
> 将来，任务可能会拦截传给 `panic!()` 的值。

但不是所有 panic 的创建都是同样的。某些情况下你可能需要终止整个程序（可能你在写一个断言，如果绊到了（if it trips），指示不可恢复的逻辑错误）；其它情况你可能想要在某些边界包含 panic（可能是来自外部的一小段畸形的输入，正好是你并行处理的， 而处理任务不能执行）。
