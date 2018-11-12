# lostutils/uq [![explain]][source] [![translate-svg]][translate-list]

<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list
[size-img]: https://packagephobia.now.sh/badge?p=Name
[size]: https://packagephobia.now.sh/result?p=Name

「 一个简单,用户友好的,`sort | uniq`替代品. 」

---

## explain 🀄️

<!-- doc-templite START generated -->
<!-- time = '2018-05-27' -->
<!-- name = 'lostutils' -->
<!-- repo = 'uq' -->
<!-- commit = '118bc2f3b1cf292afdffbc1cb4415d150b323165' -->

| 版本     | 与日期        | 最新更新   | 更多               |
| -------- | ------------- | ---------- | ------------------ |
| [commit] | ⏰ 2018-05-27 | ![version] | [源码解释][source] |

[commit]: https://github.com/lostutils/uq/tree/118bc2f3b1cf292afdffbc1cb4415d150b323165
[version]: https://img.shields.io/github/last-commit/lostutils/uq.svg

<!-- doc-templite END generated -->

### 中文 Readme

[中文](zh.md)

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---

## Cargo.toml

```toml
[package]
name = "uq"
version = "0.1.2"
authors = ["Tamir Bahar"]
repository = "https://github.com/lostutils/uq"
license = "MIT"
description = "sort | uniq alternative."
readme = "README.md"

[dependencies]
clap = "2.29.2" # rust 的命令行参数解析
fxhash = "0.2" # 哈希Map/Set

[profile.release]
lto = true

[[bin]]
name = "uq"
path = "src/main.rs"
```

看来在`src`

## src/main.rs

主要还是先从`main`主函数，开始

> 途中看看，结构/变量和 crate导入

### 目录

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [extern + use](#extern--use)
- [struct](#struct)
- [impl](#impl)
- [功能函数](#%E5%8A%9F%E8%83%BD%E5%87%BD%E6%95%B0)
  - [unique_filter](#unique_filter)
  - [unique_filter_with_cap](#unique_filter_with_cap)
  - [unique_filter_with_override](#unique_filter_with_override)
- [main](#main)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### extern + use

```rust
extern crate clap;
use clap::{App, Arg};

extern crate fxhash;
use fxhash::FxHashSet;

use std::collections::VecDeque;
use std::io::{BufRead, StdinLock, Write};
```

### struct

```rust
struct StdinReader<'a> {
    buffer: Vec<u8>,
    input: StdinLock<'a>,
}

```

### impl

```rust
impl<'a> StdinReader<'a> {
    fn new(input: StdinLock<'a>) -> Self {
        Self {
            buffer: Vec::new(),
            input: input,
        }
    }

    fn next_line(&mut self) -> Option<&Vec<u8>> {
        self.buffer.clear();
        match self.input.read_until(b'\n', &mut self.buffer) {
            Ok(0) => None,
            Ok(_) => Some(&self.buffer),
            Err(e) => panic!("Failed reading line: {}", e),
        }
    }
}

```

### 各种过滤函数

#### unique_filter

默认

```rust
fn unique_filter() -> Box<FnMut(&Vec<u8>) -> bool> {
    let mut lines = FxHashSet::default(); // Set的特性，就是唯一属性名，不重复

    // move 移动所有权 ？？？？
    Box::new(move |line| lines.insert(line.clone()))
}

```

#### unique_filter_with_cap

带容量

```rust
fn unique_filter_with_cap(capacity: usize) -> Box<FnMut(&Vec<u8>) -> bool> {
    let mut lines = FxHashSet::default();

    Box::new(move |line| {
        if lines.insert(line.clone()) {
            if lines.len() > capacity {
                panic!("Cache capacity exceeded!");
            }
            true
        } else {
            false
        }
    })
}

```

#### unique_filter_with_override



```rust
fn unique_filter_with_override(capacity: usize) -> Box<FnMut(&Vec<u8>) -> bool> {
    let mut set = FxHashSet::default();
    let mut queue = VecDeque::new();

    Box::new(move |line| {
        if set.insert(line.clone()) {
            if set.len() > capacity {
                set.remove(&queue.pop_front().unwrap());
            }

            queue.push_back(line.clone());
            true
        } else {
            false
        }
    })
}
```

### main

```rust
fn main() {
    // 命令行解析
    let matches = App::new("uq (lostutils)")
        .arg(
            Arg::with_name("capacity")
                .short("n")
                .help("Number of unique entries to remember.")
                .value_name("capacity")
                .takes_value(true), // 默认值
        )
        .arg(
            Arg::with_name("override")
                .short("r")
                .help("Override old unique entries when capacity reached.\nWhen not used, uq will die when the capacity is exceeded.")
                .requires("capacity") // 必须要有capacity
                .value_name("override")
                .takes_value(false), // 默认值
        )
        .get_matches();

    // capacity 参数，检测
    let capacity = match matches.value_of("capacity") {
        Some(n) => match n.parse::<usize>() {
            Ok(n) => Some(n),
            Err(_) => None,
        },
        None => None,
    };
    
    // 是否存在override 参数，返回对应 过滤函数
    let mut unique_filter = match (capacity, matches.is_present("override")) { // 匹配情况
        (Some(capacity), true) => unique_filter_with_override(capacity), // 若capacity有，override为true
        (Some(capacity), false) => unique_filter_with_cap(capacity), // 若capacity有，override为false
        _ => unique_filter(),
    };

    // 请记得，unique_filter 现在是 Box<**> 类型的匿名函数

    // 获得 标准输入，标准输出
    let (_in, _out) = (std::io::stdin(), std::io::stdout());
    let (input, mut output) = (_in.lock(), _out.lock()); // 先锁好

    let mut stdin_reader = StdinReader::new(input); // 读取，标准输入，返回迭代器
    while let Some(line) = stdin_reader.next_line() { // 持续迭代，由 StdinReader 自身提供的next_line函数
        if unique_filter(line) { // 过滤函数
            output.write_all(line).expect("Failed writing line");
            // expect函数，若接受了一个Err, 会发生panics,带其错误信息
        }
    }
}
```
