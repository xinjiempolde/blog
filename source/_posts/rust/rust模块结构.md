---
title: rust模块结构
date: 2024-07-28 20:04:21
tags:
    - rust
categories:
    - rust
---

> 转载自
>
> - [rust 模块组织结构 - li-peng - 博客园 (cnblogs.com)](https://www.cnblogs.com/li-peng/p/13587910.html)

`rust`有自己的规则和约定用来组织模块，比如一个包最多可以有一个库`crate`，任意多个二进制`crate`、导入文件夹内的模块的两种约定方式... 知道这些约定，就可以快速了解`rust`的模块系统。
先把一些术语说明一下：

- `包`是cargo的一个功能，当执行`cargo new xxxx`的时候就是创建了一个包。
- `crate`是二进制或者库项目。`rust`约定在`Cargo.toml`的同级目录下包含`src`目录并且包含`main.rs`文件，就是与包同名的**二进制`crate`**，如果包目录中包含`src/lib.rs`，就是与包同名的**库`crate`**。包内可以有多`crate`，多个`crates`就是一个模块的树形结构。如果一个包内同时包含`src/main.rs`和`src/lib.rs`，那么他就有两个`crate`，如果想有多个二进制`crate`，`rust`约定需要将文件放在`src/bin`目录下，每个文件就是一个单独的`crate`。
- `crate根`用来描述如何构建`crate`的文件。比如`src/main.rs`或者`src/lib.rs`就是`crate根`。**`crate根`文件将由`Cargo`传递给`rustc`来实际构建库或者二进制项目。**
- 带有`Cargo.toml`文件的包用来描述如何构建`crate`，一个包可以最多有一个库`crate`，任意多个二进制`crate`。

[github 代码地址](https://github.com/lpxxn/rust_module)

# 模块

模块以`mod`开头，下面创建了一个`say`模块

```rust
mod say {
    pub fn hello() {
        println!("Hello, world!");
    }
}
```

需要注意的是模块内，所有的项（函数、方法、结构体、枚举、模块和常量）默认都是私有的，可以用`pub`将项变为公有，上面的代码里`pub fn hello()`就是把函数`hello()`变为公有的。
子模块可以通过`super`访问父模块中所有的代码，包括私有代码。但是父模块中的代码不能访问子模块中的私有代码。

```rust
mod say {
    pub fn hello() {
        println!("Hello, world!");
    }
    fn hello_2() {
        println!("hello")
    }
    pub mod hi {
        pub fn hi_1() {
            super::hello_2();
        }
        pub fn hi_2() {
            println!("hi there");
        }
    }
}
```

## 同一文件内的模块

同一文件内的模块，最外层的`mod say`不用设置为`pub`就可以访问，但是`mod say`下面的要设置成`pub`才可以访问。

```rust
fn main() {
    // 相对路径
    say::hello();
    // 绝对路径调用
    crate::say::hello();
    
    say::hi::hi_1();
    say::hi::hi_2();
}

mod say {
    pub fn hello() {
        println!("Hello, world!");
    }
    fn hello_2() {
        println!("hello")
    }
    pub mod hi {
        pub fn hi_1() {
            super::hello_2();
        }
        pub fn hi_2() {
            println!("hi there");
        }
    }
}
```

调用模块内的方法，可以使用绝对路径以`crate`开头，也就是从`crate根`开始查找，`say`模块定义在`crate根 src/main.rs`中，所以就可以这么调用`crate::say::hello();`绝对路径类似于`Shell`中使用`/`从文件系统根开始查找文件。
相对路径以模块名开始`say`，他定义于`main()`函数相同的模块中，类似`Shell`在当前目录开始查找指定文件`say/hello`。

`mod hi`是一个嵌套模块，使用时要写比较长`say::hi::hi_2();`，可以使用`use`将名称引入作用域。

```rust
use crate::say::hi;

fn main() {
    // 相对路径
    say::hello();
    // 绝对路径调用
    crate::say::hello();

    // 不使用 use
    say::hi::hi_1();
    say::hi::hi_2();
    // 使用 use 后就可以这么调用
    hi::hi_1();
}
```

## 使用`pub use` 重导出名称

不同的模块之前使用`use`引入，默认也是私有的。如果希望调用的模块内`use`引用的模块，就要用`pub`公开，也叫**重导出**

```rust
fn main() {
    // 重导出名称
    people::hi::hi_1();
    people::hello();
    // 但是不能 
    // people::say::hello();
}

mod say {
    pub fn hello() {
        println!("Hello, world!");
    }
    fn hello_2() {
        println!("hello")
    }
    pub mod hi {
        pub fn hi_1() {
            super::hello_2();
        }
        pub fn hi_2() {
            println!("hi there");
        }
    }
}

mod people {
    // 重导出名称
    pub use crate::say::hi;
    use crate::say;
    pub fn hello() {
        say::hello();
    }
}
```

如果想都导出自己和嵌入的指定包可以用`self`，例`mod people_2` 把模块`people`和嵌套模块`info`全部导出来了。

```rust
use crate::say::hi;

fn main() {
    // 相对路径
    say::hello();
    // 绝对路径调用
    crate::say::hello();

    // 不使用 use
    say::hi::hi_1();
    say::hi::hi_2();
    // 使用 use 后就可以这么调用
    hi::hi_1();

    // 重导出名称
    people::hi::hi_1();
    people::hello();
    // 但是不能 
    // people::say::hello();

    people_2::people::hello();
    people_2::info::name();
}

mod say {
    pub fn hello() {
        println!("Hello, world!");
    }
    fn hello_2() {
        println!("hello")
    }
    pub mod hi {
        pub fn hi_1() {
            super::hello_2();
        }
        pub fn hi_2() {
            println!("hi there");
        }
    }
}

pub mod people {
    // 重导出名称
    pub use crate::say::hi;
    use crate::say;
    pub fn hello() {
        say::hello();
    }
    pub mod info {
        pub fn name() {
            println!("zhangsang");
        }
    }
}


mod people_2 {
    // 重导出名称
    pub use crate::people::{self, info};
    pub fn hello() {
        info::name();
    }
}
```

# 不同文件夹的引用

## 方式一

看一下目录结构：
![img](https://img2020.cnblogs.com/blog/342595/202008/342595-20200831092952337-240410311.png)
`rust`的约定，在目录下使用`mod.rs`将模块导出。
看一下user.rs的代码：

```rust
#[derive(Debug)]
pub struct User {
    name: String,
    age: i32
}

impl User {
    pub fn new_user(name: String, age: i32) -> User {
        User{
            name,
            age
        }
    }
    pub fn name(&self) -> &str {
        &self.name
    }
}

pub fn add(x: i32, y: i32) -> i32 {
    x + y 
}
```

然后在`mod.rs`里导出：

```rust
pub mod user;
```

在`main.rs`调用

```rust
mod user_info;
use user_info::user::User;

fn main() {
    let u1 = User::new_user(String::from("tom"), 5);
    println!("user name: {}", u1.name());
    println!("1+2: {}", user_info::user::add(1, 2));
}
```



## 方式二

看一下目录结构
![img](https://img2020.cnblogs.com/blog/342595/202008/342595-20200831093032927-706165161.png)
和上面的不同之前是。这种方式是`user_info`目录里没有`mod.rs`，但是在外面有一个`user_info.rs`
在`user_info.rs`中使用`pub mod user;`是告诉`Rust`在另一个与模块同名的文件夹内(user_info文件夹)内加载模块`user`。这也是`rust`的一个约定，但比较推荐用上面的方式。
代码和上面是一样的。
user.rs

```rust
#[derive(Debug)]
pub struct User {
    name: String,
    age: i32
}

impl User {
    pub fn new_user(name: String, age: i32) -> User {
        User{
            name,
            age
        }
    }
    pub fn name(&self) -> &str {
        &self.name
    }
}

pub fn add(x: i32, y: i32) -> i32 {
    x + y 
}
```

user_info.rs里导出

```rust
pub mod user;
```

在`main.rs`调用

```rust
mod user_info;
use user_info::user::User;

fn main() {
    let u1 = User::new_user(String::from("tom"), 5);
    println!("user name: {}", u1.name());
    println!("1+2: {}", user_info::user::add(1, 2));
}
```



# 使用外部包

使用外部包，一般就是从`crates.io`下载，当然也可以自己指写下载地点，或者使用我们本地的库，或者自建的的仓库。

## 一般方式

在`Cargo.toml`的`dependencies`下写要导入的依赖库

```ini
[dependencies]
regex = "0.1.41"
```

运行`cargo build`会从`crates.io`下载依赖库。
使用的时候，直接使用`use`引入

```python
use regex::Regex;

fn main() {
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    println!("Did our date match? {}", re.is_match("2014-01-01"));
}
```

## 指定库地址

除了`crates.io`下载依赖库，也可以自己指定地址，也可以指定`branch` `tag` `commit`，比如下面这个

```ini
[dependencies]
# 可以和包不同名，也可以同名
my_rust_lib_1={package="my_lib_1",git="ssh://git@github.com/lpxxn/my_rust_lib_1.git",tag="v0.0.2"}
```

就是从`github.com/lpxxn/my_rust_lib_1`上下载包。也可以使用`https`

```ini
my_rust_lib_1={package="my_lib_1",git="https://github.com/lpxxn/my_rust_lib_1.git",branch="master"}
```

执行`cargo build`就会自动下载，使用的时候也是一样的。

```rust
use my_rust_lib_1;
fn main() {
    println!("Hello, world!");
    println!("{}", my_rust_lib_1::add(1, 2));
    let u = my_rust_lib_1::User::new_user(String::from("tom"), 2);
    println!("user: {:#?}", u);
}
```

## 使用本地的库

我们新建一个二进制库项目

```cpp
cargo new pkg_demo_3
```

然后在`pkg_demo_3`内建一个库项目

```sql
cargo new --lib utils
```

然后就可以在 `utils`里写我们的库代码了
看一下现在的目录结构
![img](https://img2020.cnblogs.com/blog/342595/202008/342595-20200831093104024-66370552.png)
在`utils`库的`user.rs`里还是我们上面的代码

```rust
#[derive(Debug)]
pub struct User {
    name: String,
    age: i32
}

impl User {
    pub fn new_user(name: String, age: i32) -> User {
        User{
            name,
            age
        }
    }
    pub fn name(&self) -> &str {
        &self.name
    }
}

pub fn add(x: i32, y: i32) -> i32 {
    x + y 
}
```

在`lib.rs`里对`user`模块导出

```rust
pub mod user;
pub use user::User;
```

然后在我们的二进制库的`Cargo.toml`引入库

```ini
[dependencies]
utils = { path = "utils", version = "0.1.0" }
```

`path`就是库项目的路径
`main.rs`使用`use`引入就可以使用了

```rust
use utils::User;

fn main() {
    let u = User::new_user(String::from("tom"), 5);
    println!("user: {:#?}", u);
}
```

## 自建私有库

除了`crates.io`也可以自建`registrie`。这个有时间再重新写一篇帖子单独说，可以先看一下官方文档。
[官方文档:registrie](https://doc.rust-lang.org/cargo/reference/registries.html#registries)
[依赖官方文档](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)
[帖子 github 代码地址](https://github.com/lpxxn/rust_module)
