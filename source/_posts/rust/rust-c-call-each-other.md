---
title: rust-c-call-each-other
date: 2023-06-11 17:56:21
tags:
    - rust
categories:
    - linux
---

# Rust调用C语言代码

> 参考自[https://blog.csdn.net/phthon1997/article/details/126469708](https://blog.csdn.net/phthon1997/article/details/126469708)

## 创建项目

<!--more-->

使用cargo创建新项目

```shell
cargo new rust_and_c
```



## 创建build.rs

```rust
extern crate cc;

fn main() {
    cc::Build::new().file("src/double.c").compile("libdouble.a");
    cc::Build::new().file("src/third.c").compile("libthird.a");
}

```

这里的build.rs：若要创建构建脚本，我们只需在项目的根目录下添加一个 build.rs 文件即可。这样一来， Cargo 就会先编译和执行该构建脚本，然后再去构建整个项目。
导入rust的一个库叫cc，作用肯定就是和c语言调用相关啦，关于具体细节暂时可以不学。
src/double.c和src/third.c都是一会要写的两个c语言文件，指定好他们编译之后的静态库。



## 编辑Cargo.toml的内容

```toml
[package]
name = "rust_and_c"
version = "0.1.0"
build = "build.rs"

[dependencies]
libc = "0.2"

[build-dependencies]
cc = "1.0"

```

package这个地方需要添加上整个构建文件build.rs以告知需要提前构建。
build-dependencies就是关于build.rs需要的库。
dependencies是main.rs所需要的库。



## 创建C语言的文件

`src/double.c`

```c
int double_input(int input)
{
    return input * 2;
}

```



`third.c`

```c
int third_input(int input)
{
    return input * 3;
}

```



## 编写rust主函数的内容

`main.rs`

```rust
extern crate libc;

extern "C" {
    fn double_input(input: libc::c_int) -> libc::c_int;
    fn third_input(input: libc::c_int) -> libc::c_int;
}

fn main() {
    let input = 4;
    let output = unsafe { double_input(input) };
    let output2: i32 = unsafe { third_input(input) };
    println!("{} * 3 = {}", input, output2);
    println!("{} * 2 = {}", input, output);
}

```

为了在rust代码中和c代码一样的类型定义一致，这里使用了为rust准备的libc库，可以放心使用，不用管两者的类型不一致问题。
也要提前使用extern “C”来做一个声明，链接主要就是靠它来做一个类似的接口，extern告知Rust编译器这部分功能由一个外部库提供。
unsafe的作用：rust只能保证自己的代码是安全的，c语言的代码不会给你去做检查，不加unsafe是不行的，涉及到很多底层的操作。

## 整体代码结构

![http://img.singhe.art/20230611185938.png](http://img.singhe.art/20230611185938.png)



# Rust使用已存在的C语言库

> 参考[https://stackoverflow.com/questions/43826572/where-should-i-place-a-static-library-so-i-can-link-it-with-a-rust-program](https://stackoverflow.com/questions/43826572/where-should-i-place-a-static-library-so-i-can-link-it-with-a-rust-program)

## 编译出静态连接库

假设在目录`/home/singheart/Project/cpp_project`下有一个`square.c`的文件，文件内容如下:

```c
int square(int value) {
    return value * value;
}
```

 将其编译成`libsquare.a`放在同一个目录下：

```shell
gcc -c -o square.o square.c
ar -rcs libsquare.a square.o
```

## 创建build.rs

```rust
fn main() {
  println!("cargo:rustc-link-search=/home/singheart/Project/cpp_project");
}
```

## 创建main.rs

```rust
#[link(name = "square")]
extern "C" {
    fn square(val: i32) -> i32;
}

fn main() {
    let r = unsafe { square(3) };
    println!("3 squared is {}", r);
}

```

## 编写Cargo.toml

```toml
[package]
name = "rust-use-c-lib"
version = "0.1.0"
edition = "2021"
build = "build.rs"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
libc = "0.2"

[build-dependencies]
cc = "1.0"
```

