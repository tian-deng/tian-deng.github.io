---
title: 深入Rust枚举
date: 2021-01-13T11:00:00+08:00
lastmod: 2021-01-13T15:00:00+08:00
slug: peeking_inside_a_rust_enum
image: cover.png
categories:
    - Rust
tags:
    - Translation
    - Rust
    - Enum
---

>  本译文译自 [Peeking inside a Rust enum](https://fasterthanli.me/articles/peeking-inside-a-rust-enum)

---


<style>
.dialog {
    display: flex;
    flex-direction: row;
    padding-bottom: 1em;
    overflow: hidden;
}
.dialog.amos {
    flex-direction: row-reverse;
}
.dialog-head {
    align-self: flex-start;
    flex-basis: 48px;
    width: 48px;
    height: 48px;
    margin: 0.2em 0.4em;
    flex-shrink: 0;
}
.dialog-text {
    background-color: #fefdf6;
    border: 1px solid #ffeb81;
    max-width: 600px;
    align-self: flex-start;
    border-radius: 3px;
    padding: 0.4rem 0.8rem;
    overflow: hidden
}
.dialog.amos .dialog-text {
    background-color: #f6f9ff;
    border: 1px solid #cce3f9;
}
</style>
# 前言

在我最近 `Rust Q&A` `twitch` 频道里, 有些人提出了一个看起来简单的问题: 为什么 `small string` 类型, 像 `SmartString` or `SmolStr` 和 `String` 有着一样的大小, 但是 `small vec` 类型, 像 `SmallVec` 却要比 `Vec` 大?

我知道我使用了简单作为形容词, 但是事实上要了解这个问题, 我们需要一些背景知识

> 译者: 亿点点...

## 这个问题到底是什么?

我最近谈到 `Rust` [small string crates](https://fasterthanli.me/articles/small-strings-in-rust).

被这些 `crates` 导出的类型可以避免多次内存分配, 并且降低内存使用量. 让我们看一个 [smartstring](https://lib.rs/crates/smartstring) 的代码作为例子.

```rust
use smartstring::{Compact, SmartString};
use std::mem::size_of_val;

fn main() {
    let smart = SmartString::<Compact>::from("hello world");
    dbg!(size_of_val(&smart));
    let stand = String::from("hello world");
    dbg!(size_of_val(&stand));
}
```

```shell
$ cargo run -q
[src/main.rs:6] size_of_val(&smart) = 24
[src/main.rs:9] size_of_val(&stand) = 24
```

正如你看到的, 同样也是最初的问题描述的那样, 这两个类型的大小是相同的.

<div class="dialog">
    <img src="./bear.svg" class="dialog-head">
    <div class="dialog-text">
    <p>毫无疑问的, `String` 存储实际数据是在别的地方? 因为所有 `String` 类型值的大小在64位平台上都是24个字节.</p>
    </div>
</div>



是的, 这当然不是一个完整的故事, 在这个特别的例子中, `smart` 以 `inline` 的方式存储它的值(在栈上), 而标准库将值存储在堆上:

<img src="./heap-vs-not-heap.svg">

如果我们想知道每个类型到底总共使用了多少内存, 我们可以这样做:

```rust
    let smart = SmartString::<Compact>::from("hello world");
    dbg!(size_of_val(&smart));

    let stand = String::from("hello world");
    dbg!(size_of_val(&stand) + stand.capacity());
```

```shell
$ cargo run -q
[src/main.rs:6] size_of_val(&smart) = 24
[src/main.rs:9] size_of_val(&stand) + stand.capacity() = 35
```

<div class="dialog">
    <img src="./bear.svg" class="dialog-head">
    <div class="dialog-text">
        <p>好吧, 但是 - 你说 <code>String</code> 显而易见在堆中存放它的内容. 有什么方法可以正式这个说法么?
        </p>
    </div>
</div>

<div class="dialog amos">
    <img src="./author.svg" class="dialog-head">
    <div class="dialog-text">
        <p>当然有!</p>
    </div>
</div>

典型的, 在 `Linux 64-bit` 系统中栈和堆在虚拟地址中的空间相差很远, 这意味着我们如果打印字符串的元数据的地址和内容的地址, 我们就可以看到:
* `SmartString` 的元数据和内容在相邻的位置.
* `String` 的元数据和内容相距很远.

```rust
use smartstring::{Compact, SmartString};

fn main() {
    let smart = SmartString::<Compact>::from("hello world");
    let smart_meta = &smart as *const _;
    let smart_data = &smart.as_bytes()[0] as *const _;
    dbg!((smart_meta, smart_data));

    let stand = String::from("hello world");
    let stand_meta = &stand as *const _;
    let stand_data = &stand.as_bytes()[0] as *const _;
    dbg!((stand_meta, stand_data));
}
```

```shell
$ cargo run -q
[src/main.rs:7] (smart_meta, smart_data) = (
    0x00007ffce4cf4728,
    0x00007ffce4cf4729,
)
[src/main.rs:12] (stand_meta, stand_data) = (
    0x00007ffce4cf47f8,
    0x0000555f87686a60,
)
```


<div class="dialog">
    <img src="./bear.svg" class="dialog-head">
    <div class="dialog-text">
        <p>好吧, 我信了. 那么 <code>smart</code> 总是把它的数据存储在栈中吗?</p>
    </div>
</div>

不! 只有在小于24字节的时候是存储在栈上的, 就和 `String` 一样. 我们用稍微长一点的字符串来看看.

```rust
use smartstring::{Compact, SmartString};

fn main() {
    let input = "Turns out you can blame your tools *and* be a good craftsperson. Who knew?";

    let smart = SmartString::<Compact>::from(input);
    let smart_meta = &smart as *const _;
    let smart_data = &smart.as_bytes()[0] as *const _;
    dbg!((smart_meta, smart_data));

    let stand = String::from(input);
    let stand_meta = &stand as *const _;
    let stand_data = &stand.as_bytes()[0] as *const _;
    dbg!((stand_meta, stand_data));
}
```

```shell
$ cargo run -q
[src/main.rs:9] (smart_meta, smart_data) = (
    0x00007ffd460d0268,
    0x0000555f4636ca30,
)
[src/main.rs:14] (stand_meta, stand_data) = (
    0x00007ffd460d0338,
    0x0000555f4636cac0,
)
```

...然后我们就可以看到这两个类型的内容现在都放在堆上了.

<div class="dialog">
    <img src="./bear.svg" class="dialog-head">
    <div class="dialog-text">
        <p>这样, 那 <code>SmartString</code> 是否意味着买一送一的处理方式? 这和他的类型有关?</p>
        <br/>
        <p>它是怎么工作的?</p>
    </div>
</div>

<div class="dialog amos">
    <img src="./author.svg" class="dialog-head">
    <div class="dialog-text">
        <p>多么好的一个问题. 让我们谈谈枚举</p>
    </div>
</div>


## 一个单词, 多种意义

如果你有着 `C/C++/Java/C#` 的语言背景, 一个 `enum` 仅仅意味着一个 `Integer` 类型, 只是它的值有着自己的意义.

让我们看一个 `C` 的例子:

```c
#include <stdio.h>

enum Drink {
    Water,
    Soda,
    Juice,
};

int main() {
    enum Drink dri = Soda;
    printf("dri = %d\n", dri);
    printf("sizeof(dri) = %ld\n", sizeof(dri));
    printf("sizeof(int) = %ld\n", sizeof(int));
}
```

这里, 我们声明了一个枚举类型 `enum Drink`, 有三个变体, 不过仅仅是从0开始的数字, 所以我们有 `Water = 0`, `Soda = 1`, `Juice = 2`:

```shell
$ clang -Wall main.c -o main && ./main
dri = 1
sizeof(dri) = 4
sizeof(int) = 4
```

然而我不是很喜欢这个代码, 我不想要额外的限定词, 并且想要我的变量有着自己的命名空间, 但在 `C` 里面我们需要自己做这些事:

```rust
#include <stdio.h>

// this could also be done in two separate declarations
typedef enum Drink {
    Drink_Water,
    Drink_Soda,
    Drink_Juice,
} Drink;

int main() {
    Drink dri = Drink_Soda;
    printf("dri = %d\n", dri);
    printf("sizeof(dri) = %ld\n", sizeof(dri));
    printf("sizeof(int) = %ld\n", sizeof(int));
}
```

Ahhh, 好多了. 但是这里仍然有其它的东西我不是特别喜欢, 像是 `C` 的 `switch`. 下面的代码是错误的:
```c
#include <stdio.h>

typedef enum Drink {
    Drink_Water,
    Drink_Soda,
    Drink_Juice,
} Drink;

void print_drink(Drink dri) {
    switch (dri) {
        case Drink_Water:
            printf("It's water!\n");
        case Drink_Soda:
            printf("It's soda!\n");
        case Drink_Juice:
            printf("It's juice!\n");
    }
}

int main() {
    print_drink(Drink_Soda);
}
```

正确的代码是在每个 `case` 语句的结尾都使用 `break`. 这是刚起步的开发人员很早就应该了解的事情之一.