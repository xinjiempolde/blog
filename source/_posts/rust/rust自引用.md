---
title: rust自引用
date: 2024-07-28 20:04:21
tags:
    - rust
categories:
    - rust
---

> 参考：
>
> - [结构体中的自引用 - Rust语言圣经(Rust Course) (lntu.edu.cn)](https://rust.e.lntu.edu.cn/rust-course/advance/circle-self-ref/self-referential.html)

什么是自引用，如下面代码所示：

```rust
struct SelfRef<'a> {
    value: String,

    // 该引用指向上面的value
    pointer_to_value: &'a str,
}

fn main(){
    let s = "aaa".to_string();
    let v = SelfRef {
        value: s,
        pointer_to_value: &s
    };
}
```

<!--more-->

但是这个代码会报错，因为我们试图同时使用值和值的引用，最终所有权转移和借用一起发生了

rust的结构体的成员默认是私有的，但是在rust中同一个模块下定义的结构体，即使它的成员是私有的，同一个模块下的函数也能对其进行访问。。。看下面的例子，被坑了好久：

```rust

mod mymod {
    pub struct Mystruct {
        contentes : i32
    }
    impl Mystruct {
        pub fn new(con : i32) -> Mystruct {
            Mystruct {
                contentes : con,
            }
        }
        
    }
}
struct Mystruct {
    contentes : i32
}

fn main() {
    let mystruct = Mystruct { contentes: 1 };
    println!("{}", mystruct.contentes);

    // 错误的,不能访问其他mod的struct成员的私有变量
    let Mystruct = mymod::Mystruct { contentes: 2 };

    // 这样子是可以的
    let mystruct = mymod::Mystruct::new(3);
}

```



Option的map用法

```rust
pub fn map<U, F>(self, f: F) -> Option<U> where    F: FnOnce(T) -> U,
```

通过将函数应用于包含的值，将 `Option<T>` 映射到 `Option<U>`。

```rust
let maybe_some_string = Some(String::from("Hello, World!"));
// `Option::map` takes self *by value*, consuming `maybe_some_string`
let maybe_some_len = maybe_some_string.map(|s| s.len());

assert_eq!(maybe_some_len, Some(13));
```



unwrap_or 和 unwrap_or_else 都是用于从 Result (Option也可以？)对象中获取值的宏。

当 Result 对象是 Ok 时，两者都会返回 Ok 中的值。但是当 Result 对象是 Err 时，两者的行为不同：

unwrap_or 将返回一个默认值。这个默认值是宏的参数，在调用 unwrap_or 时就已经确定了。

unwrap_or_else 将调用一个闭包，并返回闭包的结果。这个闭包是宏的参数，在调用 unwrap_or_else 时就已经确定了。

所以，当你想要在 Err 时使用固定的默认值时，就可以使用 unwrap_or；而当你想要在 Err 时使用可变的值时，就可以使用 unwrap_or_else。

示例代码：

```rust
let x: Result<i32, &amp;str> = Err("error message");

// 使用 unwrap_or 返回默认值
let y = x.unwrap_or(0);

// 使用 unwrap_or_else 返回闭包的结果
let z = x.unwrap_or_else(|| {
    println!("error message: {}", x.unwrap_err());
    0
```
