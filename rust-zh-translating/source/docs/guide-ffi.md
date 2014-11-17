% Rust 外部函數接口指南

# 簡介

本篇使用 [snappy](https://github.com/google/snappy)
解/壓縮函數庫爲示例介紹如何綁定外部代碼。Rust 目前尚無法直接調用 C++ 函數庫，但 snappy 包含了 C 語言型接口 (可參見文檔：
[`snappy-c.h`](https://github.com/google/snappy/blob/master/snappy-c.h)).

下面是一個可以在安裝了 snappy 函數庫的環境裏能夠正常編譯的最小化示例：

~~~~no_run
extern crate libc;
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("一個 100 字節緩衝的最大壓縮長度: {}", x);
}
~~~~

`extern` 區塊中存放了外部庫中的函數簽名列表，在上面示例中該區塊默認爲 C ABI(應用二進制接口) 風格。 `#[link(...)]` 屬性註釋(下文有時亦簡稱爲：「註解」)則用於指示鏈接器對應使用 snappy 函數庫以完成符號連接。

C 函數庫開放的接口常常的非線程安全的，也由於指針可以進行多種變換，幾乎所有的函數都會使用未檢驗指針型參數來接納任何可能的入參，再加上野指針會溢出 Rust 的內存安全模型。故外部函數被編譯系統假定爲不安全代碼，所以在調用這些函數的時候需要使用 `unsafe {}` 區域包裹起來，以向編譯器聲明該區塊內的代碼由開發人員保證其安全性，方能使編譯得以繼續。

Rust 編譯器無法檢測出這些外部函數聲明中參數類型是否正確，只有聲明正確才能在運行時期正確地完成綁定。

現在嘗試在 `extern` 區塊內覆蓋到整個 snappy 函數庫提供的 API(應用程序編程接口):

~~~~no_run
extern crate libc;
use libc::{c_int, size_t};

#[link(name = "snappy")]
extern {
    fn snappy_compress(input: *const u8,
                       input_length: size_t,
                       compressed: *mut u8,
                       compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(compressed: *const u8,
                         compressed_length: size_t,
                         uncompressed: *mut u8,
                         uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(compressed: *const u8,
                                  compressed_length: size_t,
                                  result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(compressed: *const u8,
                                         compressed_length: size_t) -> c_int;
}
# fn main() {}
~~~~

# 創建安全接口

原始的 C API 一般需要封裝處理以保證內存安全，同時也便於將接口思考層次轉化爲諸如向量器(Vectors)之類的高級概念。函數庫可以選擇僅暴露安全、高層次接口，而隱藏那些非安全的內部細節。


下文源碼中接口函數使用向量器作爲緩存器對外部調用進行了包裝，而其內部則使用了 `slice::raw` 模塊將 Rust 向量器處理成指向內存區塊的指針。這裏大致介紹一下向量器，Rust 將向量器的設計保證爲一個連續的內存區塊。其 length 屬性代表所保留的元素數量， capacity 表示申領的內存所能容納元素的總數。length 屬性值小於等於 capacity 屬性值。

~~~~
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_validate_compressed_buffer(_: *const u8, _: size_t) -> c_int { 0 }
# fn main() {}
pub fn validate_compressed_buffer(src: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len() as size_t) == 0
    }
}
~~~~

上文源碼中 `validate_compressed_buffer` 包裝函數內部使用了 `unsafe` 區塊，但其通過自身函數簽名約定確保了對其本身的調用是安全的，並且所有入參數據也都不是非安全的(回憶一下，最初關於調用 `unsafe` 內容時必需遵守的約定條款)。

`snappy_compress` 與 `snappy_uncompress` 兩函數更複雜一些，因爲它們申領內存以保存輸出結果。

`snappy_max_compressed_length` 函數則可用於計算出創建向量器時所需的最大空限值，用以保證向量器可完全容納壓縮結果數據。接着就可以將向量器傳遞給 `snappy_compress` 函數作爲輸出參數了。向量器會在壓縮過程結束後被設定成正確長度。

~~~~
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_compress(a: *const u8, b: size_t, c: *mut u8,
#                           d: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(a: size_t) -> size_t { a }
# fn main() {}
pub fn compress(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen as uint);
        let pdst = dst.as_mut_ptr();

        snappy_compress(psrc, srclen, pdst, &mut dstlen);
        dst.set_len(dstlen as uint);
        dst
    }
}
~~~~

解壓過程是相似的過程，因爲 snappy 壓縮格式中存儲了其未壓縮尺寸， `snappy_uncompressed_length` 函數則可檢索出壓縮所需的緩衝器大小。

~~~~
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t) -> c_int { 0 }
# fn main() {}
pub fn uncompress(src: &[u8]) -> Option<Vec<u8>> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen: size_t = 0;
        snappy_uncompressed_length(psrc, srclen, &mut dstlen);

        let mut dst = Vec::with_capacity(dstlen as uint);
        let pdst = dst.as_mut_ptr();

        if snappy_uncompress(psrc, srclen, pdst, &mut dstlen) == 0 {
            dst.set_len(dstlen as uint);
            Some(dst)
        } else {
            None // 錯誤的 SNAPPY 輸入數據
        }
    }
}
~~~~

本篇使用的示例代碼已存儲為 [GitHub 項目](https://github.com/thestinger/rust-snappy) 方便讀者參考。

# 堆棧區管理

Rust 任務默認運行在一塊「超大堆棧」上。該堆棧目前被實現為在內存地址空間中預留的一塊超大尺寸分割區，並按照任務的實際需要完成惰性映射工作。當調用外部函數庫時，外部代碼也被加載到了相同的堆棧區中以便調用。由於編譯系統假定該堆棧區足以同時容納外部代碼，所以整個調用過程並不存在額外的堆棧切換機制。

Rust 計畫未來在每個 Rust 堆棧尾部追加一個防禦頁(撰寫本文時尚未實現)以改善現狀。任何 Rust 函數都不會觸及到該防禦頁(參見 LLVM 關於 `__morestack` 的說明)。實現這個非映射用頁意在防禦可能出現的從 C 代碼無限遞歸而溢出到其他 Rust 堆棧區的情況。當觸及防禦頁時，此任務過程將被中斷並返回錯誤訊息。

從用戶層面來講，這意味著普通外部函數使用過程不必再做額外的處理工作了。C 堆棧將與 Rust 堆棧自然地交錯存放，而且該堆棧塊大小足以應付兩方交互。即便真的出現檢測出需要更大尺寸堆棧的情況，任務裝配 API 也會提供相應的函數用以控制裝配任務所需的堆棧大小。

# Destructors

外部庫往往並不關心從調用代碼處簽屬資源後相關的管理工作。當遇到這種情況時，一定要利用 Rust's destructors 及時釋放權屬資源以防止程序崩潰，保證代碼的健壯與安全。

# 從 C 代碼回調 Rust 函數

一些外部函式庫會要求調用方提供回調方式以回報當前狀態或交換數據，為此 Rust 也提供了對應的解決方案。只需要將回調函數標記為 extern ，經由編譯系統正確的調用轉換，便可在 C 代碼中調用。

可以通過註冊的方式將回調函數發往外部，接著便可從外部調用該函數了。

以下是一個基本示例：

Rust 源碼：

~~~~no_run
extern fn callback(a: i32) {
    println!("I'm called from C with value {0}", a);
}

#[link(name = "extlib")]
extern {
   fn register_callback(cb: extern fn(i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    unsafe {
        register_callback(callback);
        trigger_callback(); // 觸發回調
    }
}
~~~~

C 源碼：

~~~~c
typedef void (*rust_callback)(int32_t);
rust_callback cb;

int32_t register_callback(rust_callback callback) {
    cb = callback;
    return 1;
}

void trigger_callback() {
    cb(7); // 將調用 Rust 中的暴露方法 callback(7) 
}
~~~~

示例裡 Rust 代碼中 `main()` 函數調用了外部庫函數 `trigger_callback()` ，該函數則通過內部存儲的函數指針調用了 Rust 代碼中定義的回調函數 `callback()` 。


## 對應回調函數 Rust 對象

前面的示例中展示了從 C 代碼中調用暴露函數的方法，然而其常常伴隨着回調同時亦傳送 Rust 對象作爲交換的需求。該對象一般來說是對所需 C 對象的再包裝形式。

通過使用非安全指針可將該對象下傳給外部 C 代碼。外部代碼則可通過註冊過程使用指針保存對該 Rust 對象的引用。這一過程使得外部代碼可以通過非安全訪問方式引用到 Rust 對象。

Rust 源碼:

~~~~no_run

#[repr(C)]
struct RustObject {
    a: i32,
    // 其它成員
}

extern "C" fn callback(target: *mut RustObject, a: i32) {
    println!("這裏從 C 代碼調用處獲得數據： {0}", a);
    unsafe {
        // 使用從回調處接收到的數據更新 RustObject 對象
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
   fn register_callback(target: *mut RustObject,
                        cb: extern fn(*mut RustObject, i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    // Create the object that will be referenced in the callback
    let mut rust_object = box RustObject { a: 5 };

    unsafe {
        register_callback(&mut *rust_object, callback);
        trigger_callback();
    }
}
~~~~

C 源碼:

~~~~c
typedef void (*rust_callback)(void*, int32_t);
void* cb_target;
rust_callback cb;

int32_t register_callback(void* callback_target, rust_callback callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void trigger_callback() {
    cb(cb_target, 7); // 將調用 Rust 代碼暴露的函數 callback(&rustObject, 7)
}
~~~~

## 異步調用

在前面的示例中，外部 C 函數庫直接觸發了回調函數請求。整個回調過程被控制在線程內以 Rust 到 C 再到 Rust 的順序切換執行。需要瞭解的是最後執行的函數回調與引發回調請求的函數是運行在同一線程(也是同一任務)上的。

但當外部函數庫轉而使用不同線程請求回調函數時，事情就變得複雜了許多。若不使用適當的同步機制，回調函數在上下文訪問的 Rust 域內數據結構會變得尤其不安全。除了像 mutexes 之類的傳統同步機制外，使用 Rust 中的頻道 (位於 `std::comm`) 也能夠完成數據從請求回調函數的 C 線程到 Rust 任務上下文中的傳遞工作。

如異步回調需要使用 Rust 地址空間內的某一對象，則必須確保在該對象銷毀後，外部庫不會發起新請求。可以在外部庫中設計回調函數反註冊接口，並通過在對象的析構操作中主動註銷回調函數，來確保對象銷毀後外部庫不會再執行新的回調請求。

# 鏈接

`link` 註解位於 `extern` 上方，用於指示編譯系統鏈接到本地函數庫的方式。目前 link 註解包含兩個元素可使用：

* `#[link(name = "foo")]`
* `#[link(name = "foo", kind = "bar")]`

在這兩個示例中 `foo` 指出了要鏈接到的本地函數庫的名稱，第二示例中的 `bar` 則向編譯器指示了要連接到的本地函數庫類型。目前已知的本地函數庫類型有三種：

* 動態(Dynamic) - `#[link(name = "readline")]`
* 靜態(Static) - `#[link(name = "my_build_dependency", kind = "static")]`
* 框架(Frameworks) - `#[link(name = "CoreFoundation", kind = "framework")]`

請注意 `框架` 這一類型僅適用於 Mac OS X 平臺。

`kind` 的設值不同，也意味着本地函數庫參與鏈接過程的方式不同。從編譯角度來講，Rust 編譯器能夠產生兩種不同風格的輸出：部件態的靜態庫 (rlib/staticlib) 和 終態的二進制庫 (dylib/binary)。本地動態函數庫和框架庫都會包含到最終發布產品中，而靜態庫則不會。

以下簡略介紹兩者的使用場景：

* 本地編譯依賴。有時在編寫 Rust 代碼時會需要黏合一些 C/C++ 代碼。但另外再分發 C/C++ 代碼的二進制當只是在徒增煩惱。遇到這種情況時，可以將 C/C++ 代碼打包到歸檔函數庫(一般是 .a 後綴名)中，接著就可以利用 `#[link(name = "foo", kind = "static")]` 註釋聲明依賴關係(假定聲稱的歸檔庫是 `libfoo.a`)中。

無論代碼最終生成為何種風格，歸檔函數庫中的代碼都會包含在最終輸出裏，同時也就避免了最終發布時額外分發本地靜態庫(那些 C/C++ 代碼的最終發布檔)的麻煩。

* 常規動態依賴。一類被稱為：公共系統函數庫的庫文件對於多數系統來說都是默認包含的 (比如 `readline`) ，這些函數庫也通常不提供靜態庫版本。若在創作工具箱時引用了此類函數庫功能，那麼當生成為開發部件庫(rlib 文件)時，是不會連接到該函數庫的，如果將輸出的 rlib 文件引入到最終目標文件(比如 二進制文件)的話，則會連接到函數庫。

在 OSX 平臺下，框架庫與動態函數庫行為一致。

## `link_args` 屬性註解

Rust 中亦通過 `link_args` 註解方式提供了自定義鏈接過程的方法。該註解會作用於 `extern` 區塊，可用於指定在編譯鏈接過程中傳遞給鏈接器的原始參數。以下給出一個簡短的例子：

~~~ no_run
#![feature(link_args)]

#[link_args = "-foo -bar -baz"]
extern {}
# fn main() {}
~~~

要注意的是，由於該功能並非爲鏈接過程正式認可的方式，所以當前被隱藏於 `feature(link_args)` 特性之下。更深層次的原因是，編譯程序 rustc 目前暫時還是使用外部傳參的方式與系統鏈接器進行交互，才使得註解可以向鏈接器傳遞額外的命令行參數。未來 rustc 可能會使用 LLVM 直接鏈接本地庫，到那時 `link_args` 註解則會失效。

這裏還是強調一下最好 *不要* 使用該註解，還是推薦使用更爲正式的 `#[link(...)]` 註解。

# 非安全區域

像是取用非安全指針內容，亦或是調用被標記爲非安全的函數之類的操作僅被允許在非安全區塊中進行。非安全區塊用於隔離非安全代碼，同時也編譯器保證其中的非安全代碼不會溢出該區塊。

另一方面也可以定義非安全函數暴露到外部。非安全函數的定義方法如下：

~~~~
unsafe fn kaboom(ptr: *const int) -> int { *ptr }
~~~~

該函數僅可從 `unsafe` 區塊中代碼或者其它 `unsafe` 函數調用。

# 訪問外部全局變量

外部接口庫常常會暴露全局變量以提供如追蹤全局狀態之類的便利。當需要訪問這些變量時，需要在 `extern` 區塊中使用 `static` 關鍵字創建變量：

~~~no_run
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("當前環境中安裝的 readline 版本爲： {} 。",
             rl_readline_version as int);
}
~~~

若還需要修改外部接口提供的全局變量進行修改，可再追加 `mut` 關鍵字完成可修改聲明。

~~~no_run
extern crate libc;
use std::ptr;

#[link(name = "readline")]
extern {
    static mut rl_prompt: *const libc::c_char;
}

fn main() {
    "[my-awesome-shell] $".with_c_str(|buf| {
        unsafe { rl_prompt = buf; }
        // get a line, process it
        unsafe { rl_prompt = ptr::null(); }
    });
}
~~~

# 外部調用轉換

多數外部代碼都會選擇暴露 C ABI，Rust 編譯系統默認也會在調用外部代碼處使用 C 風格調用轉換。但還有一部分外部函數，特別要注意的是 Windows API 則使用了其它的調用轉換。Rust 提供了指示編譯器使用調用轉換風格的方法：

~~~~
extern crate libc;

#[cfg(all(target_os = "win32", target_arch = "x86"))]
#[link(name = "kernel32")]
#[allow(non_snake_case)]
extern "stdcall" {
    fn SetEnvironmentVariableA(n: *const u8, v: *const u8) -> libc::c_int;
}
# fn main() { }
~~~~

此設定應用於整個 `extern` 代碼塊。ABI 支援列表如下：

* `stdcall`
* `aapcs`
* `cdecl`
* `fastcall`
* `Rust`
* `rust-intrinsic`
* `system`
* `C`
* `win64`

除了 `system` 這一項外，多數備選條目的意義都很明確。 `system` 選項會自行選取與目標函數庫相匹配的 ABI 設定。簡單打個比方，在 x86 平台的 Windows 系統裡，會自動匹配 `stdcall` 風格。而 x86_64 平台的 Windows 系統使用了 `C` 風格，`system` 設定也能自行適應。所以對於前述示例，能夠不限於 x86 平台，而在所有 Windows 系統中均使用 `extern "system" { ... }` 格式編寫外部函數接口定義區塊。

# 與外部代碼互通

對於應用了 `#[repr(C)]` 註解的 `結構體`，編譯器會使用與 C 平臺相容的內存佈局方式。使用`#[repr(C, packed)]` 註解則可指定結構體成員字節對齊方式。 `#[repr(C)]` 同樣適用於 enum 。

Rust 權屬箱(owned boxes，亦即：`Box<T>`)使用指向容納對象的非空指針作為句柄。由於其由內部分配機負責管理，所以不應以手工方式進行創建。因為假定了其非空指針特性，所以引用可以安全、快速地獲取到被裝箱類型。注意儘量只在必要的情況下才適用原始指針 (`*`) ，因為編譯器能對其所做的推斷工作有限，若意外打破了對象出借檢測或者變更約定的話，就無法確保代碼安全性了。

向量器(Vectors)與字符串(strings)使用了相同的基本內存而已方式，其與 C 協作的相關工具函數分別存放在 `vec` 及 `str` 模塊下。儘管字符串並非以 `\0` 結尾，但如果需要的話可以通過  `c_str::to_c_str` 函數將其轉化為 C 平臺風格的 `NULL` 結尾字符串。

在標準庫方面，`libc` 模塊中針對 C 標準庫定義了大量類型別名和函數定義。需要注意的是，Rust 默認情況下不會鏈接到 `libc` 和 `libm` 工具箱。

# 可空指針優化方案

包含 引用 (`&T`, `&mut T`), 收納箱 (`Box<T>`), 函數指針 (`extern "abi" fn()`) 在內的類型都被設定了非 `null` 特性。當與 C 對接的時候，指針常常有可能爲 `null`。針對於這種特殊情況，一個定制的枚舉類型樣板很適合作爲「可空指針優化方案」，該枚舉僅包含兩個變量，一個用表示空狀態，另一個則用於存放值域。當實例化此枚舉類型後，非空類型用於表示非空指針，空類型則用於表示空指針。依此，可以使用 `Option<extern "C" fn(c_int) -> c_int>` 描述一個 C ABI 風格的可空函數指針。
