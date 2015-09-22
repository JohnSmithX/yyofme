+++
title = "Rust的Deref coercion"
tags = ["Rust", "trait", "Deref"]
date = "2015-09-20"
categories = [
  "Rust"
]
+++

## Rust的Deref coercion

### 了解强制多态

> 编译程序通过语义操作，把操作对象的类型强行加以变换，以符合函数或操作符的要求。程序设计语言中基本类型的大多数操作符，在发生不同类型的数据进行混合运算时，编译程序一般都会进行强制多态。程序员也可以显示地进行强制多态的操作(Casting)。举个例子，比如，int+double，编译系统一般会把int转换为double，然后执行double+double运算，这个int->double的转换，就实现了强制多态，即可是隐式的，也可显式转换。

简单来说当变量遇到某种操作时，发生了类型转换。

### [Deref](http://doc.rust-lang.org/nightly/std/ops/trait.Deref.html) trait

`Deref`是`Rust`的一个`trait`。`trait`相当于其它语言的接口或者协议。`Deref`的一般用来重载`*`解引用运算符。

```rust
use std::ops::Deref;

struct Example {
	value: i32,
}

impl Deref for Example {
	type Target = i32;
    fn deref(&self) -> &i32 {
    	&self.value
    }
}


fn main() {
	let x = Example{ value: 5 };
    assert_eq!(5, *x); // 解引用会执行 Deref::deref(&self) 返回一个i32类型
}
```

### [Deref coercion](https://doc.rust-lang.org/book/deref-coercions.html)

**deref coercions** 解引用强制多态在`Rust`中的规则：假如一个类型`U`实现了`Deref`，那么 `&U`类型将自动转化成`&Self::Target`。(`Target`具体类型)

```rust
struct U<T>(T);

impl<T> Deref for U<T> {
	type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
	// 这里`&U`类型被自动转换成了`&Self::Target`，也就是`&i32`
	let _x: &i32 = &U::<i32>(5);
}
```



