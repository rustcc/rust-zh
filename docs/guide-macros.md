% Rust 宏指南

<div class="unstable-feature">
<b>警告：</b> 当前宏调用还存在许多问题，它们是如何与环境交互的，如何在其定义外面使用的。宏定义很可能在未来稍微的改变。基于此，它们隐藏在<code>宏规则(macro_rules)</code> <a href="reference.html#compiler-features">特性属性</a>后面。
</div>

# 简介

函数是程序员用来构建抽象的主要工具。然而有时希望在编译时语法上抽象而不是运行时的值。宏提供语法抽象。关于这点如何有用的例子，考虑下面两个代码片段，两个都是对输入的模式匹配且在一种情况提前返回而在其它情况啥都不干：

~~~~
# enum T { SpecialA(uint), SpecialB(uint) }
# fn f() -> uint {
# let input_1 = T::SpecialA(0);
# let input_2 = T::SpecialA(0);
match input_1 {
    T::SpecialA(x) => { return x; }
    _ => {}
}
// ...
match input_2 {
    T::SpecialB(x) => { return x; }
    _ => {}
}
# return 0u;
# }
~~~~

如果重复很多次的话，这段程序就显得令人讨厌了。不过没有函数能够捕捉它的功能使其将重复抽象掉。然而 Rust 的宏系统可以消除这些重复。宏是轻量级自定义的语法扩展，它们本身的定义使用 `macro_rules!` 的语法扩展。下面的 `early_return` 宏捕获上面程序中的模式：

~~~~
# #![feature(macro_rules)]
# enum T { SpecialA(uint), SpecialB(uint) }
# fn f() -> uint {
# let input_1 = T::SpecialA(0);
# let input_2 = T::SpecialA(0);
macro_rules! early_return(
    ($inp:expr $sp:path) => ( // invoke it like `(input_5 SpecialE)`
        match $inp {
            $sp(x) => { return x; }
            _ => {}
        }
    );
)
// ...
early_return!(input_1 T::SpecialA);
// ...
early_return!(input_2 T::SpecialB);
# return 0;
# }
# fn main() {}
~~~~

宏是以模式匹配风格定义的：在上面的例子中，`=>` 左边出现的文本 `($inp:expr $sp:ident)` 是*宏调用语法(macro invocation syntax)*，模式标记如何写宏调用。`=>` 右边以 `match $inp` 开头的文本是*宏展开语法(macro transcription syntax)*：宏展开成什么。

# 调用语法（invocation syntax）

宏调用语法说明了宏参数的语法。它出现在宏定义中 `=>` 的左边，遵守下面的规则：

1. 它必须由括号包起来
2. `$` 有特殊的含义（描述见下面）
3. 它包含的括号 `()`，`[]`， `{}` 必须匹配，例如 `([)` 是禁止的

除此之外，调用语法的形式很自由。

要接收一段 Rust 程序作为参数，在 `$` 后跟一个名字（在右边使用），接着是 `:`，然后是*片段说明符(fragment specifier)*。片段说明符表示匹配的片段种类，最常见的片段说明符是：

* `ident`（标识符，指一个变量或条目。例如：`f`, `x`, `foo`）
* `expr`（表达式，例如：`2 + 2`; `if true then { 1 } else { 2 }`; `f(42)`）
* `ty`（类型，例如：`int`, `Vec<(char, String)>`, `&T`）
* `pat`（模式，通常出现在 `match` 中或申明左边，例如：`Some(t)`; `(17, 'a')`; `_`）
* `block`（一系列动作，例如：`{ log(error, "hi"); return 12; }`）

语法分析器按字面解释任何不以 `$` 开头的符号，应用 Rust 正常的标记化。

所以尽管 `($x:ident -> (($e:expr)))` 过度的复杂，仍然会指定一个宏，能够像这样调用：`my_macro!(i->(( 2+2 )))`

## 调用位置（invocation location）

宏调用会替代（因此展开成）一个表达式、条目或陈述。
Rust 语法分析器会将宏调用解析为”占位符(placeholder)“，不管对其中的哪个，三个非终止符(nonterminal)就适合于这个位置。

展开时，宏输出会解析为这三个非终止符所代表的（内容）。意思是单个宏，例如展开成一个条目或一个表达式，依赖于其参数（并且如果在其位置上调用错误的参数就会导致语法错误）。尽管这点听起来太过动态化了，但在某些情况下仍然是有用的。

# 展开语法（transcription syntax）

`=>` 右边与左边遵循同样的规则，除了 `$` 后面需要跟一个语法片段的名字用来将其展开成宏展开；其类型无需重复。

右边必须以定界符闭合，展开器会忽略该定界符。因此 `() => ((1,2,3))` 宏会展开成元组表达式，`() => (1,2,3)` 宏会展开成语法错误（因为展开器将右边的括号解释为定界符，而 `1,2,3` 本身不是 Rust 有效的表达式）。

除了允许 `$name`（和下面讨论的 `$(...)*`），宏定义右边是 Rust 普通的语法。特别地，宏调用（包括当前定义的宏调用）允许是表达式，陈述和条目位置。然而程序其它的东西都不会被宏系统检查或执行；执行仍然需要等到运行时。

## 插入位置（interpolation location）

插入 `$argument_name` 会出现在任何与片段说明符一致的地方（即如果被指定为 `ident`，就可以用在任何标识符允许的地方）。

# 多样性（multiplicity）

## 调用（invocation）

回到导引例子（motivating example），回想一下 `early_return` 展开成一个 `match`，会在当 `match` 匹配到由 `early_return` 第二个参数提供的“特殊情况”标识符时 `return`，而其它情况什么也不干。现在假设我们想写一个能够处理可变数量的“特殊”情况的 `early_return`。

宏定义中， `=>` 右边的 `$(...)*` 语法接受0个或更多内容。它与正则表达式中的 `*` 算符很像。它也支持分隔符（逗号分隔的列表可以写成 `$(...),`），和 `+` 替代 `*` 代表“至少一次”。


~~~~
# #![feature(macro_rules)]
# enum T { SpecialA(uint),SpecialB(uint),SpecialC(uint),SpecialD(uint)}
# fn f() -> uint {
# let input_1 = T::SpecialA(0);
# let input_2 = T::SpecialA(0);
macro_rules! early_return(
    ($inp:expr, [ $($sp:path)|+ ]) => (
        match $inp {
            $(
                $sp(x) => { return x; }
            )+
            _ => {}
        }
    );
)
// ...
early_return!(input_1, [T::SpecialA|T::SpecialC|T::SpecialD]);
// ...
early_return!(input_2, [T::SpecialB]);
# return 0;
# }
# fn main() {}
~~~~

### 展开（transcription）

如上面例子演示的那样，`$(...)*` 在宏定义右边也是有效的。展开中 `*` 的行为，特别是在多个 `*` 嵌套情况，并且牵涉到多个不同的名字时，起初看起来像魔法并且不直观（译注：原文是 intuitive，个人觉得应该是 non-intuitive）。解释它们的系统称为“通过示例的宏（Macro By Example）”，两个要记住的规则是（1）`$(...)*` 的行为是对于包含的所有 `$name` 齐步地遍历重复的一层；（2）每个 `$name` 必须至少与其匹配的 `$(...)*` 一样多。如果更多，就会适当地重复。

## 解析限制（parsing limitations）

由于技术上的原因，宏解析器处理语法片段有两个限制：

1. 解析器总是尽可能多地解析 Rust 语法片段。例如如果上面例子中的 `early_return!` 语法中略去了逗号，`input_1 [` 会被解释为数组索引的起点。实际上调用这个宏是不可能的。
2. 解析器必须在到达 `$name:fragment_specifier` 申明时消除所有歧义。这个限制会导致当申明在开头或紧跟 `$(...)*` 之后时出现解析错误。例如，`$($t:ty)* $e:expr` 语法总是会解析失败，因为解析器会被强制在解析 `t` 和 `e` 之间选择。将调用语法改成要求前面是一个不同的符号可以解决这个问题。在上面的例子中，`$(T $t:ty)* E $e:exp` 就解决了这个问题。

# 宏参数模式匹配（macro argument pattern matching）

## 动机（Motivation）

现在考虑下面的程序：

~~~~
# #![feature(macro_rules)]
# enum T1 { Good1(T2, uint), Bad1}
# struct T2 { body: T3 }
# enum T3 { Good2(uint), Bad2}
# fn f(x: T1) -> uint {
match x {
    T1::Good1(g1, val) => {
        match g1.body {
            T3::Good2(result) => {
                // complicated stuff goes here
                return result + val;
            },
            _ => panic!("Didn't get good_2")
        }
    }
    _ => return 0 // default value
}
# }
# fn main() {}
~~~~

所有复杂的的玩意儿都是深度缩进的，并且错误处理程序与失败的匹配是分开的。我们希望写一个宏执行匹配，但用一个更适应问题的语法。下面的宏可以解决这个问题：

~~~~
# #![feature(macro_rules)]
macro_rules! biased_match (
    // special case: `let (x) = ...` is illegal, so use `let x = ...` instead
    ( ($e:expr) ~ ($p:pat) else $err:stmt ;
      binds $bind_res:ident
    ) => (
        let $bind_res = match $e {
            $p => ( $bind_res ),
            _ => { $err }
        };
    );
    // more than one name; use a tuple
    ( ($e:expr) ~ ($p:pat) else $err:stmt ;
      binds $( $bind_res:ident ),*
    ) => (
        let ( $( $bind_res ),* ) = match $e {
            $p => ( $( $bind_res ),* ),
            _ => { $err }
        };
    )
)

# enum T1 { Good1(T2, uint), Bad1}
# struct T2 { body: T3 }
# enum T3 { Good2(uint), Bad2}
# fn f(x: T1) -> uint {
biased_match!((x)       ~ (T1::Good1(g1, val)) else { return 0 };
              binds g1, val )
biased_match!((g1.body) ~ (T3::Good2(result) )
                  else { panic!("Didn't get good_2") };
              binds result )
// complicated stuff goes here
return result + val;
# }
# fn main() {}
~~~~

这个解决了缩进问题，但如果有大量像这样的链式匹配，我们可能宁愿写单个简单的宏调用。我们想要的输入模式很清楚：

~~~~
# #![feature(macro_rules)]
# fn main() {}
# macro_rules! b(
    ( $( ($e:expr) ~ ($p:pat) else $err:stmt ; )*
      binds $( $bind_res:ident ),*
    )
# => (0))
~~~~

然而，不可能直接展开嵌套匹配陈述。但有一个解决方案。

## 递归方法写宏（the recursive approach to macro writing）

宏可以接受多个不同的输入语法，第一个成功匹配到实际参数的宏调用是赢家。

在上面示例中，我们想要写一个递归宏来一个一个地处理分号结束的行。所以我们想要下面的输入模式：

~~~~
# #![feature(macro_rules)]
# macro_rules! b(
    ( binds $( $bind_res:ident ),* )
# => (0))
# fn main() {}
~~~~

而且：

~~~~
# #![feature(macro_rules)]
# fn main() {}
# macro_rules! b(
    (    ($e     :expr) ~ ($p     :pat) else $err     :stmt ;
      $( ($e_rest:expr) ~ ($p_rest:pat) else $err_rest:stmt ; )*
      binds  $( $bind_res:ident ),*
    )
# => (0))
~~~~

结果宏看起来像这样。注意 `biased_match!` 和 `biased_match_rec!` 中间出现的分隔只是因为我们有一段外部的语法（`let`），我们只想展开一次。

~~~~
# #![feature(macro_rules)]
# fn main() {

macro_rules! biased_match_rec (
    // Handle the first layer
    (   ($e     :expr) ~ ($p     :pat) else $err     :stmt ;
     $( ($e_rest:expr) ~ ($p_rest:pat) else $err_rest:stmt ; )*
     binds $( $bind_res:ident ),*
    ) => (
        match $e {
            $p => {
                // Recursively handle the next layer
                biased_match_rec!($( ($e_rest) ~ ($p_rest) else $err_rest ; )*
                                  binds $( $bind_res ),*
                )
            }
            _ => { $err }
        }
    );
    // Produce the requested values
    ( binds $( $bind_res:ident ),* ) => ( ($( $bind_res ),*) )
)

// Wrap the whole thing in a `let`.
macro_rules! biased_match (
    // special case: `let (x) = ...` is illegal, so use `let x = ...` instead
    ( $( ($e:expr) ~ ($p:pat) else $err:stmt ; )*
      binds $bind_res:ident
    ) => (
        let $bind_res = biased_match_rec!(
            $( ($e) ~ ($p) else $err ; )*
            binds $bind_res
        );
    );
    // more than one name: use a tuple
    ( $( ($e:expr) ~ ($p:pat) else $err:stmt ; )*
      binds  $( $bind_res:ident ),*
    ) => (
        let ( $( $bind_res ),* ) = biased_match_rec!(
            $( ($e) ~ ($p) else $err ; )*
            binds $( $bind_res ),*
        );
    )
)


# enum T1 { Good1(T2, uint), Bad1}
# struct T2 { body: T3 }
# enum T3 { Good2(uint), Bad2}
# fn f(x: T1) -> uint {
biased_match!(
    (x)       ~ (T1::Good1(g1, val)) else { return 0 };
    (g1.body) ~ (T3::Good2(result) ) else { panic!("Didn't get Good2") };
    binds val, result )
// complicated stuff goes here
return result + val;
# }
# }
~~~~

这个技术适用于许多情况，当不能立即展开一个结果时。结果程序在某些方面与普通函数式语言相似，但有一些重要的不同。

第一个不同很重要，但也容易忘记：`macro_rules!` 的展开那边（右边）是字面语法（literal syntax），只能在运行时执行。如果一段展开语法本身不出现在另一个宏调用中，就会成为最终程序的一部分。如果是在一个宏调用中（例如，递归调用 `biased_match_rec!`），就有机会影响展开，但只通过尝试模式匹配过程。

第二个相关的不同是宏求值顺序与普通编程相比的向后感知。给一个调用 `m1!(m2!())`，展开器首先展开 `m1!`，给它一个字面语法 `m2!()` 作为输入。如果无变化地将参数展开到适当的位置（特别地，不是作为参数给另一个宏调用），然后展开器就会求值 `m2!()`（与其它任何产生的宏调用 `m1!(m2!())`）。

# 卫生（hygiene）

为了避免冲突，Rut 实现了[卫生宏](http://en.wikipedia.org/wiki/Hygienic_macro)。

作为一个示例，`loop` 和 `for-loop` 标签（生存期指南中讨论的）不会冲突。下面的程序只会打印 "Hello!" 一次：

~~~
#![feature(macro_rules)]

macro_rules! loop_x (
    ($e: expr) => (
        // $e will not interact with this 'x
        'x: loop {
            println!("Hello!");
            $e
        }
    );
)

fn main() {
    'x: loop {
        loop_x!(break 'x);
        println!("I am never printed.");
    }
}
~~~

两个 `'x` 名字不会冲突，冲突的话会导致循环打印 "I am never printed" 并且永远运行下去。

# 作用域和宏导入/导出

宏占据了一个单独的全局名字空间。与 Rust 的模块系统和箱子交互某种意义上是很复杂的。

宏定义和展开都发生在单深度优先，词法顺序遍历箱子源文件。所以同一个模块中定义在模块作用域的宏在其后的代码中都是可见的，包括其后任何子条目 `mod` 体。

如果一个模块有 `macro_escape` 属性，其宏在父模块的子条目 `mod` 后也是可见的。如果父模块也有 `macro_escape`，那么该宏在祖模块的父条目 `mod` 后也是可见的，以此类推。

`macro_export` 属性控制不同的箱子之间的可见性，而不依赖于 `macro_escape` 。任何带有 `macro_export` 属性的 `macro_rules!` 定义，在该箱子被 `phase(plugin)` 加载时在其它箱子中是可见的。当前导入的箱子还不能够控制哪个宏被导入。


一个例子：

```rust
# #![feature(macro_rules)]
macro_rules! m1 (() => (()))

// visible here: m1

mod foo {
    // visible here: m1

    #[macro_export]
    macro_rules! m2 (() => (()))

    // visible here: m1, m2
}

// visible here: m1

macro_rules! m3 (() => (()))

// visible here: m1, m3

#[macro_escape]
mod bar {
    // visible here: m1, m3

    macro_rules! m4 (() => (()))

    // visible here: m1, m3, m4
}

// visible here: m1, m3, m4
# fn main() { }
```

当这个库通过 `#[phase(plugin)] extern crate` 加载时，只有 `m2` 会被导入。

# 最后的说明

当前实现的宏需要用心。当错误发生在宏内部时，即使是常规的语法错误也会更加难调试，生成的代码中由解析问题导致的错误可能非常棘手。调用 `log_syntax!` 宏可以帮助阐明中间状态，调用 `trace_macros!(true)` 会自动将中间状态打印出来，而将 `--pretty expanded` 标记作为命令行参数传递给编译器会显示展开的结果。

如果 Rust 的宏系统不能做你需要的事情，你可能想写一个[编译器插件](guide-plugin.html)。与 `macro_rules!` 宏相比，这个显然需要更多的工作，稳定性也低很多，并且调试的警告变成r原来10倍。作为交换，你得到了在编译器中跑任意 Rust 程序的灵活性。语法扩展插件也因这个原因有时也被称作“过程宏（procedural macros）”。
