% 用 Rust 编写非安全和底层代码

# 简介

Rust 的设计目标之一就是在 CPU 和操作系统底层之上提供安全抽象，但是有些时候我们确实需要在底层编写代码。本文将对 Rust 的非安全（unsafe）功能的危险与能力进行介绍。

Rust 提供了 `unsafe { ... }` 代码块。该代码块类似于一种“逃生舱”，让程序员躲开编译器的检查，得以完成非常自由的操作，例如：

- 去引用 (dereferencing) 操作 [原始指针](#原始指针)
- 通过外部函数调用 (FFI) 调用外部库函数 ([在 FFI 指南介绍](guide-ffi.html))
- 二进制级别的类型转换（`transmute`，也称为 "reinterpret cast"）
- [嵌入汇编](#嵌入汇编)

请注意，`unsafe` 代码块并没有让编译器放松对 `&` 操作的生命期检查，也没有对借出数据解冻。

使用 `unsafe` 相当于程序员告诉编译器：“我比你知道得更多”，而事实上，程序员自己也必须非常确定这些非安全代码到底在做什么。通常情况下，每一个程序员都应该尽量减少非安全代码在总程序中所占的比例；推荐的方式是，对外提供安全的接口来隐藏内部的非安全代码，使得非安全代码块尽量精简。

> **注意**: Rust 语言的底层细节仍然在变化中，并且也不会对“前向兼容”有任何承诺。甚至有可能某些改变并不会造成编译错误，但实际上其语义却变化了（导致不确定的行为）。因此需要我们非常小心地处理非安全代码。

# 指针 (Pointers)

## 引用 (References)

Rust 最大的特性之一就是内存安全。这个特性一部分是由 [生命周期管理](guide-lifetimes.html) 来达成的。编译器确保每一个 `&` 引用都是有效的，并且从来都不会指向已经被释放的内存。

编译器对 `&` 的检查带来了巨大的好处，但是同时也限制了我们如何使用 `&`。例如，`&` 并不等价于 C 指针，因此不能在外部函数调用中（FFI）被直接当做指针来使用。另外基于内存安全的需要，不可变(`&`)和可变 (`&mut`) 引用还有一些别名和冻结方面的限制，

特别要指出，如果你拥有 `&T` 这个引用，那么 `T` 的内容绝对不能通过 `&T` 或者其他引用来变更。而有一些标准库中的类型，如 `Cell` 和 `RefCell` 则提供了内部变更能力，让编译器跳过编译时检查，替之以运行时检查。（译者注：`Cell` 和 `RefCell` 正是通过 `unsafe` 块躲过了编译时检查，使得可以直接修改 `T` 的内容。）

`&mut` 则有不同的限制：如果在某个特定时刻，`&mut T` 引用指向了某个对象，那么这个 `&mut` 引用一定是整个程序中唯一能够访问到该对象的“途径”。也就是说，`&mut` 不能和其他到该对象的引用同时存在。（译者注：这两段话就是 Rust 编译器对引用的最基本安全限制，而 `unsafe` 则正是为了绕过这个限制。）

不正确地使用 `unsafe` 来躲避编译时检查可能会导致不可预期的结果。例如：下面的代码创建了两个 `&mut` 指针的别名，可能导致非预期的结果。

```
use std::mem;
let mut x: u8 = 1;

let ref_1: &mut u8 = &mut x;
let ref_2: &mut u8 = unsafe { mem::transmute(&mut *ref_1) };

// oops, ref_1 and ref_2 point to the same piece of data (x) and are
// both usable
*ref_1 = 10;
*ref_2 = 20;
```

译者注：该例中 `ref_1` 在安全代码中创建，这没问题。`ref_2` 以非安全的方式也获得了对 `x` 的 `&mut` 引用，编译器的检查被跳过去了。那么后续 `*ref_1 = 10` 和 `*ref_2 = 10` 实际都修改了 `x` 的值，因此会导致 `*ref_1` 最终也变成了 `20`。这可能不是作者预期的结果。

## 原始指针 (Raw pointers)

Rust 提供了两种附加的 “原始指针”类型：`*const T` 和 `*mut T`。它们大致等价于 C 语言的 `const T*` 和 `T*`；事实上，它们最常用的地方也是外部函数调用 (FFI)，用来和 C 语言库对接。

和 Rust 语言和系统库提供的其他指针类型相比，原始指针是最不被保障的指针类型。例如：

- 它们并不被保证指向有效内存，甚至不确保其值是非空的（不像 `Box` 和 `&`）;
- 它们不像 `Box` 会被自动清理，因此你必须实现自己的资源管理。
- 它们指向的内容是“简单旧数据”（plain-old-data）。换句话说，这些数据没有 Rust 的“拥有权”的概念，自然也不用通过“借出”来传递“拥有权”，这点也和 `Box` 不同。所以编译器也就不会帮你避免那种“释放后使用”的 Bug。
- 这些指针天生就是可传递的（Sendable，译者注：指可以直接在 Tasks 之间被传递）（当然指向的内容也是可以被传递的）。编译器在线程安全方面对原始指针帮不上什么忙，例如，两个线程可能未经任何同步操作就同时访问 `*mut int`。
- 它们缺乏任何形式的生命周期。和 `&` 不同，编译器没有任何方式来确认“迷途指针”。
- 除了不能直接修改 `*const T` 类型原始指针指向的内容，编译器无法确保对原始指针的起别名、以及修改原始指针的变更属性（译者注：加不加 `mut` 限定）是否符合 Rust 的规则。

幸运的是，非安全代码也给程序员带来了补偿：更少的保障意味着更少的限制。限制缺失让原始指针适合于实现类似于标准库中的“智能指针”和“向量” (Vectors) 类型。例如：`*` 一个原始指针（译者注：`*` 表示去引用操作 (dereferencing)， 例如对原始指针 `ref_1` 进行去引用的写法为 `*ref_1`）被允许别名化，因此可以被用于一些需要共享拥有权的场景，例如实现支持引用计数或者垃圾回收机制的指针，或者自己实现线程安全的共享内存类型（标准库中的`Rc`（引用计数） 和 `Arc`（自动引用计数） 类型就都是完全用 Rust 实现的）。

一般情况下，使用原始指针有两件事需要注意（`unsafe { ... }`）：

- 去引用 (dereferencing): 由于没有任何限制，因此可以给去引用赋任何值（例如前例的 `*ref_1 = 10; *ref_2 = 20;`）：因此有可能会导致程序崩溃、访问了未初始化内存、或者正常地读到了数据。
- 当使用指针的 `offset` [操作](#intrinsics)（或者.offset 方法）时，只有所谓的“界内” (指针偏移计算结果在原始对象的内存范围内) 运算才可能获得期待的结果。

后面这条允许编译器可以更有效地优化代码。我们也可以看到，真正地 *建立* 一个原始指针并不是非安全的，而且也不能被转换为整型。

译者注：关于 dereferencing，也就是 `*ptr`，我都翻译为“去引用”，也许有更好的译法。

### 引用和原始指针

在运行时，指向相同数据的原始指针 `*` 和引用的内部表现形式都是一样的。事实上，安全代码中的 `&T` 方式的引用在内部会被强制转变为 `*const T` 原始指针和它的`mut` 变种（`&T` 和 `&mut T` 分别会被转变为 `value as *const T` 和 `value as *mut T`）。

反过来看，将 `*const` 转变为 `&` 则可能是不安全的。由于一旦原始指针转换到 `&T` 之后，编译器就认为该 `&T` 引用就将一直是安全有效的，是已经被确保过的。因此程序员在转换前 *必须* 要确保原始指针 `*const T` 一定会指向 `T` 类型的有效实例，也要遵循该引用的别名和变更法则。

建议的转换方法为：

```
let i: u32 = 1;
// explicit cast
let p_imm: *const u32 = &i as *const u32;
let mut m: u32 = 2;
// implicit coercion
let p_mut: *mut u32 = &mut m;

unsafe {
    let ref_imm: &u32 = &*p_imm;
    let ref_mut: &mut u32 = &mut *p_mut;
}
```

相对于 `transmute`， `&*x` 是更推荐的去引用方式。`transmute` 的强大超过了我们的需要，显然，更多的限制可以让我们更少地犯错；上例中，使用 `&*x` 起码要求 `x` 是一个指针，而 `transmute` 则接受任何类型作为参数。

## 如何让非安全代码更安全

有一些不同方法为非安全代码暴露安全接口：

- 将原始指针私有保存（不要把原始指针作为公有结构的公有字段），这样你就能集中控制所有对原始指针的读写操作。
- 大量使用  `assert!()` 断言：由于你不能借助于编译器和类型系统来保证你的 `unsafe` 代码是正确的，那么就需要通过 `assert!()` 在运行时来确保正确性。
- 实现 `Drop` 方法，这样可以在析构时释放资源。并且使用 RAII (Resource Acquisition Is Initialization 编程方式。译者注：构造同时初始化资源，析构同时释放资源）。这将减少调用者手工管理内存的需求，并且确保自动清理工作总能执行，即使在任务异常时 (task panic) 。
- 确保原始指针指向的数据在合适的时间都能被正确地释放。

作为例子，我们重新实现了基于 `malloc` 和 `free` 的装箱操作。Rust 的“移动”语义和生命期确保了我们的新实现和 `Box` 类型一样安全。

```
#![feature(unsafe_destructor)]

extern crate libc;
use libc::{c_void, size_t, malloc, free};
use std::mem;
use std::ptr;

// 定义一个封装器封装一个外部传入的句柄
// Unique<T> 实现了和 Box<T> 相同的语义
pub struct Unique<T> {
    // 成员为原始可变更指针，指向相关对象
    ptr: *mut T
}

// Implement methods for creating and using the values in the box.

// NB: 为了简化代码和避免出错, 我们要求 T 拥有 Send trait
// 译者注：可以跨 Task 传递，拥有 `static 生命期
// ( Box<T> 则没有这个限制 )。
impl<T: Send> Unique<T> {
    pub fn new(value: T) -> Unique<T> {
        unsafe {
            let ptr = malloc(mem::size_of::<T>() as size_t) as *mut T;
            // 我们 *需要* 一个有效的指针
            assert!(!ptr.is_null());
            // `*ptr` 还没有被初始化。如果用 `*ptr = value` 的方式赋值
            // 那么将首先试图销毁 `*ptr`的内容，然后再用 value 的值 `覆盖它`
            // 而下面的方法将直接用新值覆盖 `ptr` 指向的内存
            ptr::write(&mut *ptr, value);
            Unique{ptr: ptr}
        }
    }

    // 这里指定返回值（对 `self.ptr` 内容的引用）和传入的 `self` (`Unique<T>`)
    // 有相同的生命周期 'r。
    // 这使得 `&*x` 和 Box<T> 的语义完全相同
    pub fn borrow<'r>(&'r self) -> &'r T {
        // By construction, self.ptr is valid
        unsafe { &*self.ptr }
    }

    // 这里指定返回值（对 `&mut *self.ptr` 内容的引用）和传入的 `mut self` (`Unique<T>`)
    // 有相同的生命周期 'r。
    // 这使得 `&*x` 和 Box<T> 的语义完全相同
    pub fn borrow_mut<'r>(&'r mut self) -> &'r mut T {
        unsafe { &mut *self.ptr }
    }
}

// 作为编写安全代码的重要元素，我们为 Unique<T> 结构提供析构函数，
// 让结构本身能够管理原始指针：当结构实例超出作用域时原始指针可以被自动释放。
//
// 请注意：这是一个非安全的析构函数，因为一般情况下 rustc 不允许析构函数
// 被关联到参数化的类型，因为对于受控空间（Managed Box）来说，这是“坏”的交互方式。
// 但是由于我们之前指定了 Send，这对我们的实现不是一个问题。
// 请注意，我们使用 `#[unsafe_destructor]` 标注来告诉编译器允许使用非安全析构函数
#[unsafe_destructor]
impl<T: Send> Drop for Unique<T> {
    fn drop(&mut self) {
        unsafe {
            // 下面这句将对象从指针指向的内存中复制到栈上
            // 随后就会被 rust 自动释放掉（离开当前栈作用域的时候）
            ptr::read(self.ptr as *const T);

            // 最后将我们之前用 malloc 分配的内存释放掉
            free(self.ptr as *mut c_void)
        }
    }
}

// rust 的 `Box` 和我们的实现之间的比较
fn main() {
    {
        let mut x = box 5i;
        *x = 10;
    } // `x` 在这里被释放

    {
        let mut y = Unique::new(5i);
        *y.borrow_mut() = 10;
    } // `y` 在这里被释放
}
```

请注意，构造 `Unique` 的唯一方法就是使用 `new` 函数，并且该函数确保内部指针的有效性和私有性。这两个 `borrow` 方法是安全的，因为编译器静态保障对象不会在建立前和析构之后被使用（除非你自己使用了 `unsafe` 代码块来访问）。

# 嵌入汇编 (Inline assembly)

如果追寻极度性能和底层操作能力，程序员可能希望直接控制 CPU。Rust 通过 `asm!` 宏支持使用嵌入汇编。这个语法和 GCC & Clang 的语法大致一样：

```ignore
asm!(assembly template
   : output operands
   : input operands
   : clobbers
   : options
   );
```

使用嵌入汇编需要在 crate 级别中标记 `#![feature(asm)]`，而且也必须在 `unsafe` 代码块中。

> **注意**: 这里给出的汇编代码是以 x86/x86-64 汇编为例的, Rust
> 也支持其他平台的汇编。

## 汇编模板 (Assembly template)

`assembly template` 是 `asm!` 宏调用参数中必须提供的，其值也必须是引号中的字符串。

```
#![feature(asm)]

#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn foo() {
    unsafe {
        asm!("NOP");
    }
}

// other platforms
#[cfg(not(any(target_arch = "x86", target_arch = "x86_64")))]
fn foo() { /* ... */ }

fn main() {
    // ...
    foo();
    // ...
}
```

译者注：上例中在 x86 平台下用汇编实现了 `foo`，而其他平台则用 Rust 语言来实现 `foo`。

(后文中的 `feature(asm)` 和 `#[cfg]` 将被省略。)

输出操作符 (Output operands)、输入操作符 (Input operands)、寄存器列表 (Clobbers) 和一些选项参数 (options) 都是可选的，但是你必须添加正确数量的 `:` 来跳过它们。

```
# fn main() { unsafe {
asm!("xor %eax, %eax"
    :
    :
    : "eax"
   );
# } }
```

空白字符不重要。

```
# fn main() { unsafe {
asm!("xor %eax, %eax" ::: "eax");
# } }
```

## 操作符 (Operands)

输入和输出操作符遵循相同的格式："constraints1"(expr1), "constraints2"(expr2), ..."`。输出操作符必须是一个可变更的左值 (lvalues):

```
fn add(a: int, b: int) -> int {
    let mut c = 0;
    unsafe {
        asm!("add $2, $0"
             : "=r"(c)
             : "0"(a), "r"(b)
             );
    }
    c
}

fn main() {
    assert_eq!(add(3, 14159), 14162)
}
```

## 寄存器列表 (Clobbers)

有些汇编指令会修改寄存器的值，因此我们通过 Clobbers 通知编译器哪些寄存器的值可能在汇编代码执行后会被修改。

```
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
// Put the value 0x200 in eax
asm!("mov $$0x200, %eax" : /* no outputs */ : /* no inputs */ : "eax");
# } }
```

输入和输出寄存器不需要列在这里，因为这是缺省的约定。除此以外，其他汇编代码中任何隐含或显示用到的寄存器都应该在这里被列出来。

如果汇编代码修改了条件码寄存器 (condition code register)，那么需要把 `cc` 加到 Clobbers 列表中。类似地，如果会汇编代码更改了某块内存的内容，那么应该把 `memory` 加入到列表中。

## 选项 (Options)

最后面的 `options` 是特别用于 Rust 的。格式为逗号分隔的字符串 (例如：`:"foo", "bar", "baz"`)。它被用于指定嵌入汇编用到的特定信息。

当前有效的 options 是：

1. **volatile** - 相当于 gcc/clang 中的 `__asm__ __volatile__ (...)`。
2. **alignstack** - 有些指令需要特定形式的栈对齐 (例如：SSE)，用这个选项让编译器插入栈对齐代码。
3. **intel** - 用 intel 的语法替代缺省的 AT&T 语法。

# 不使用标准库

缺省情况下，`std` 标准库被链接到每一个 Rust crate，在某些情况下我们可能不希望如此 (译者注：你可能需要做一个最精简的 Rust 程序，C 语言级别的尺寸)。这可以通过在 crate 层级设置 `#![no_std]` 属性避免 `rustc` 链接标准库。

```ignore
// a minimal library
#![crate_type="lib"]
#![no_std]
```

很显然，除了库的问题之外还需要考虑其他问题：你可以为一个可执行程序指定 `#[no_std]`，然后用以下两种方式之一设定程序的入口：为你的入口函数设定 `#[start]` 属性，或者覆盖 Rust 为 C `main` 函数生成的 shim (译者注：shim 就不强行翻译了，可以理解成 “垫片” 或者转换程序?)。

下面的 `start` 函数被标记为  `#[start]`，意味着该函数将收到以 C 语言的 `main` 函数格式的命令行参数：

```
#![no_std]
#![feature(lang_items)]

// Pull in the system libc library for what crt0.o likely requires
extern crate libc;

// Entry point for this program
#[start]
fn start(_argc: int, _argv: *const *const u8) -> int {
    0
}

// These functions and traits are used by the compiler, but not
// for a bare-bones hello world. These are normally
// provided by libstd.
#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
```

为了避免编译器为你插入一个 `main` 函数作为 shim，你必须另外指定 `#![no_main]` 属性来手工禁止编译器这一行为。然后要确保符号表中的入口带正确的 ABI (Application Binary Interface) 和名称，这需要你指定 `#[no_mangle]` 来避免编译器对该函数进行 “name mangling” (译者注：name mangling 是指编译器编译函数时，其名称为名字+参数+类型等的复杂表示，概念来源于 C++ 编译器。如果要对外暴露简单的 C 格式函数，则需要避免 name mangling)：

```ignore
#![no_std]
#![no_main]
#![feature(lang_items)]

extern crate libc;

#[no_mangle] // ensure that this symbol is called `main` in the output
pub extern fn main(argc: int, argv: *const *const u8) -> int {
    0
}

#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
```

编译器会假设一些特点的函数（目前是三个）在可执行文件中已经存在。一般情况下，这些函数是在标准库中被提供。如果不用标准库，则需要你自己来定义。

这三个函数的第一个是 `stack_exhausted`。该函数将在堆栈溢出被检测到时被调用。这个函数被调到是有一系列的前提的，如果 Task 没有维护自己的堆栈上限寄存器 (stack limit register )，那么该 Task 将拥有一个 “无限大的堆栈” ("infinite stack")，那么这个函数将不会被触发。

第二个函数是 `eh_personality`，用于编译器的故障机制。这个函数通常被映射到 GCC 的 personality 函数 (更多信息请参见 [libstd implementation](std/rt/unwind/index.html))。如果 Crate 不会触发 Panic (指未捕获的异常)，那么也就不会触发这个函数。三个函数的最后一个 `panic_fmt` 也是用于编译器的异常机制的。

## 使用核心库 (libcore)

> **请注意**: 核心库的结构目前还不稳定，所以请尽量用标准库来替代。

借助于上面的技术，我们可以获得一个最小的能运行 Rust 代码的执行程序。标准库提供了很好的功能来提升 Rust 程序员的生产力。如果标准库不能满足你的特别要求，那么可以试试用 [libcore](core/index.html) 来替代。

核心库 (libcore) 的依赖很少，而且也比标准库易于移植。核心库 (libcore) 中包含大部分必要的功能来撰写符合 Rust 风格和效率的代码。

如下例程序，计算两个向量 (向量值来自于外部的 C 代码) 的点积：

```
#![no_std]
#![feature(globs)]
#![feature(lang_items)]

# extern crate libc;
extern crate core;

use core::prelude::*;

use core::mem;

#[no_mangle]
pub extern fn dot_product(a: *const u32, a_len: u32,
                          b: *const u32, b_len: u32) -> u32 {
    use core::raw::Slice;

    // Convert the provided arrays into Rust slices.
    // The core::raw module guarantees that the Slice
    // structure has the same memory layout as a &[T]
    // slice.
    //
    // This is an unsafe operation because the compiler
    // cannot tell the pointers are valid.
    let (a_slice, b_slice): (&[u32], &[u32]) = unsafe {
        mem::transmute((
            Slice { data: a, len: a_len as uint },
            Slice { data: b, len: b_len as uint },
        ))
    };

    // Iterate over the slices, collecting the result
    let mut ret = 0;
    for (i, j) in a_slice.iter().zip(b_slice.iter()) {
        ret += (*i) * (*j);
    }
    return ret;
}

#[lang = "panic_fmt"]
extern fn panic_fmt(args: &core::fmt::Arguments,
                       file: &str,
                       line: uint) -> ! {
    loop {}
}

#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
# #[start] fn start(argc: int, argv: *const *const u8) -> int { 0 }
# fn main() {}
```

注意，这里定义了 lang item，`panic_fmt`。这个函数必须由核心库的使用者来定义，因为核心库虽然声明了它，却没有定义其实现。`panic_fmt` 是该 crate 中的 panic，用户必须确保其永远不返回。

核心库的设计是跨平台的，试图让所有环境和平台都能发挥出 Rust 的能力。而其他库，如 liballoc，则进一步为核心库添加平台相关的功能，同时保持比标准库更高的可移植性。

# 和编译器内部进行交互

> **注意**: 这一节是专门针对 `rustc` 编译器的；这部分的规范可能永远不会完全确定，
> 其细节也可能在不同的实现中差别很大(甚至是 `rustc` 自己的各个版本的实现差别都可能很大)。
>
> 另外，这里只是一个概述；最好的“文档”是其函数定义以及 `std` 中的对它们的用法。

目前，Rust 语言有两种正交的机制来允许库的编写者和编译器进行交互：

- 内联函数 (intrinsics), 直接在函数体内生成 LLVM IR (LLVM 中间语言)，而不是由编译产生。
- lang-items, 为库中的 functions, types 和 traits 添加 `#[lang]` 属性，以替换编译器的缺省实现。

## 内联函数 (Intrinsics)

译者注：内联函数 (Intrinsics) 就是指示编译器通过执行你的函数体来直接生成 LLVM IR (LLVM 中间语言)。

> **请注意**: intrinsics 的接口永远是不稳定的, 因此建议你使用稳定的 libcore 接口，而不是直接使用 intrinsics。

有些从 FFI 引入的函数，指定了 `rust-intrinsic` ABI (Application Binary Interface)。例如，

希望能够在类型间 (types) 进行转换，并且进行高效的指针运算，那么开发者需要按照如下方式进行声明：

如果你需要引入特定的外部函数 (FFI) 以实现高效率的类型 (types) 转换 (transmute) 和指针运算，而这些函数是 intrinsics，那么可以按照下例来声明：

```
# #![feature(intrinsics)]
# fn main() {}

extern "rust-intrinsic" {
    fn transmute<T, U>(x: T) -> U;

    fn offset<T>(dst: *const T, offset: int) -> *const T;
}
```

译者注：真实的 `transmute` 和 `offset` 实现是以 intrinsic 方式实现的，也就是说，这两个函数实际上是生成了类型转换和指针偏移计算的 LLVM 中间语言，LLVM 编译器则直接用这些中间代码来生成平台相关的机器码，这样的方式对于如此底层而常用的功能是非常高效（运行时的效率）的。

正如其他外部函数调用 (FFI)，这些代码都是 `unsafe` 调用。

## Lang items

> **请注意**: Rust 生成的 Crate 将提供缺省的 lang items 实现，
> 而且由于 lang items 自身的接口和定义是不稳定的，因此建议你尽量使用编译器缺省的生成的 lang items。

`rustc` 编译器支持一些插件化的操作，使得某些功能并一定在语言编译器中写死，而交给特定的库来实现。你可以通过一些事先约定的标志通知编译器，后续的代码将用于实现哪些编译器功能。这些标志的格式为  `#[lang="..."]`。`...` 值代表着某些编译器需要用到的功能项，这些功能项就是 "lang items"。

例如，`Box` 指针需要两种不同的 lang items，一个用于分配内存 (`lang="exchange_malloc"`)，另一个用于释放内存 (`lang="exchange_free"`)。下面的例子 (不依赖标准库) 借助简单的 `malloc` 和 `free` 自己实现了 `Box` 语法糖。

译者注：Rust 标准库对 `Box` 的实现当然不会这么简单，但是通过下面的例子，读者可以发现自己拥有怎样的 “权力” 来改变编译器的缺省实现。

```
#![no_std]
#![feature(lang_items)]

extern crate libc;

extern {
    fn abort() -> !;
}

#[lang="exchange_malloc"]
unsafe fn allocate(size: uint, _align: uint) -> *mut u8 {
    let p = libc::malloc(size as libc::size_t) as *mut u8;

    // malloc failed
    if p as uint == 0 {
        abort();
    }

    p
}
#[lang="exchange_free"]
unsafe fn deallocate(ptr: *mut u8, _size: uint, _align: uint) {
    libc::free(ptr as *mut libc::c_void)
}

#[start]
fn main(argc: int, argv: *const *const u8) -> int {
    let x = box 1i;

    0
}

#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
```

请注意对 `abort` 的使用：`exchange_malloc` lang item 期待返回的是一个有效的指针，因此在 `allocate` 实现内部有一个有效性检查，如果不通过就 `abort`。

其他 lang item 包括：

- 通过 traits 实现的重载操作: `==`、`<`、去引用 (`*`) 、 `+` 等相关 traits 都可以被 lang item 标记；分别对应为 `eq`、`ord`、`deref` 和 `add`。
- 栈展开 (stack unwinding) 和通用故障：对应 `eh_personality`、 `fail`
  和 `fail_bounds_checks` lang items。
- `std::kinds` 中的 traits，用于声明类型的 “种类” (kinds)。对应的 lang item 是 `send`、`sync` 和`copy` (译者注：不确定 kind 在 Rust 中应该翻译为什么，其含义为用来声明某个类型是不是可传递、可复制等)。
- `std::kinds::markers` 中声明的标记类型及其变种，对应的 lang items 为 `covariant_type`、
  `contravariant_lifetime`、 `no_sync_bound` 等。

Lang items 是被编译器延迟载入的 (loaded lazily)，举例来说，如果代码中没有用到 `Box`，那么也就不会载入 `exchange_malloc` 和 `exchange_free` 的所定义标记的函数。而当有需要时，如果 `rustc` 在当前 crate 及其依赖的 crate 中找不到 lang item 的相关定义时，则会报错。