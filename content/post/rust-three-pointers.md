+++
title = "Rust的三种指针"
tags = ["Rust", "指针"]
date = "2015-09-09"
categories = [
  "Rust"
]
+++

## Rust的三种指针

### Summary

#### 他们都有谁？

* 所有权指针 unique pointers
* 借贷指针 borrowed pointers
* 原始指针 raw pointers

#### 有什么不同？

* 所有权（不是指针）：绑定一个值并拥有所有权 `let a = 5;` a有5的所有权
* 所有权指针：和所有权类似但所绑定的值是一个指针类型 `let a = Box::new(5);`
* 借贷指针：引用一个现有的值，但是并不分配内存，只是一个地址的别名
* 原始指针：和c/c++指针类似

#### Tips：

* 数据可变性遵从指针的可变性`let a = Box::new(5);`不可变，`let mut a = Box::new(5);`可变
* 在任何时候只有一个**所有权指**针指向同一内存
* **所有权指针**和**原始指针必**须解引用才能它们，但在Rust中，方法调用会自动解引用

    ```rust
    struct Person;

    impl Person {
        fn say_hello(self) {
            println!("hello");
        }
    }
    let a = Box::new(Person);
    let b: Person = *a;// 需要解引用
    // or
    let a.say_hello();// 自动解引用
    ```

* 值的**所有权**可以通过重新绑定来转移，但只能存在一个**owner**，当值不在有绑定时，会被释放

### 所有权指针

##### 让我们创建一个所有权指针：

```rust
let a: Box<i32> = Box::new(5);
```

`Box::new(5)`会在堆中分配一个`i32`类型值为`5`的空间，`a`是一个指向它的`所有权指针`，`a`在桟上并且是唯一的。

##### 让我们看看它的有哪些特性：

```rust
// 我们不能通过`*a = 6`来改变它的值，因为它的可变性遵从`a`的可变性
let a: Box<i32> = Box::new(5);
*a = 6; // Fail
// 所以
let mut a: Box<i32> = Box::new(5);
*a = 6; // OK

/*==========================================================*/
// 所有权指针必须通过解引用来使用，但调用方法会自动解引用
struct Person;
impl Person {
    fn say_hello(self) {
        println!("hello");
    }
}
let a = Box::new(Person);
let b: Person = *a;// 解引用
// or
let a.say_hello();// 自动解引用

/*==========================================================*/
//在一般情况下，Rust具有移动而不是复制的语义，但原始类型具有复制的语义
let a = 5;
let b = a;//`a`是`i32`类型属于原始类型，所以`b`持有`a`的拷贝。详：`Copy` 和 `Clone` trail

// but
let a = Box::new(5);
let b = a;// 这里`a`持有的所有权指针转移给了`b`
```

### 借贷指针

我们使用`&`操作符创建一个借来的引用，用`*`来解引用。**借贷指针**不分配内存，类似**别名**，自动解引用规则也应用于**借贷指针**。

```rust
let a: &i32 = &5;// `a`为值`5`的一个借贷引用
let b: i32 = *a;// 使用 `*` 解引用
```

##### 不可变的借贷指针不是唯一的：

```rust
let a: i32 = 5;
let b: &i32 = &a;
let c: &i32 = b;
let d: &i32 = b;
println!("values {} {} {}", *b, *c, *d);
```

**注：可以看出借贷指针类型具有具有复制的语义**

##### 可变的借贷指针是唯一的：

```rust
let mut a: i32 = 5;// `a`为可变的绑定
let b: &mut i32 = &mut a;// b为一个可变借贷指针，获得一个可变绑定的可变引用
*b = 6;// OK 因为`b`是可变借贷指针
let c: &mut i32 = &mut a;// Fail 可变的借贷引用同一时间只能存在一个
// 所以
{
	let b: &mut i32 = &mut a;// 独立的组用域
}
let c: &mut i32 = &mut a;// OK `b`已经被销毁
```

##### 变量的可变性和值的可变性：

```rust
let mut a: i32 = 5;
let mut b: i32 = 6;// 可变绑定`a`和`b`
let mut c: &mut i32 = &mut a;// `c`的类型为可变借贷指针，本身是可变的
c = &mut b;          // Ok `c`可以引用不同的可变绑定
```

### 原始指针

**原始指针**表现和`c/c++`类似，在`Rust`中原始指针又名**不安全指针**，因为**原始指针**可能是一个**空指针**或者**野指针**，解引用它们是不安全的。

```rust
let a: *const i32 = &5;// 通过明确的制定变量为指针类型来得到一个原始指针
let b: *mut i32 = &5;// 原始指针两种形式 `*const T`和`*mut T`
```

`*const T`为不可变的**原始指针**，`*mut T`为可变的**原始指针**，可以解引用后改变引用的值。

```rust
let a: *mut i32 = &5;
unsafe {
	*a = 6;// 可变原始指针，可以改变引用的值
}
```

##### 原始指针需要在`unsafe`块中解引用:

```rust
let a: *const i32 = &5;
unsafe {
	println!("{:?}", *a);
}

// Fail
let a: *const i32 = &5;
println!("{:?}", *a);// Fail 需要`unsafe`块解引用
```

##### Example:

```rust
use std::rc::Rc;

fn bar(x: Rc<i32>) { }
fn baz(x: &i32) { }

fn main() {
    let a: Rc<i32> = Rc::new(45);
    bar(x.clone());   // 此处使用`clone`时要显示的告诉使用者复制了一个原始指针，并且引用计数加一
    baz(&*x);         // 没有在`unsafe`中解引用，是因为已经在`Rc`中封装，并不会增加引用计数
    println!("{}", 100 - *x);
}
```

### 小结

> &emsp;&emsp;`Rust`的安全来自严格的类型使用规范，只要搞懂了`Rust`如何通过强大的指针系统来实现安全的所有权系统就能在编译器编译通过的情况下，能够按照预期在执行，更不用担心各种潜在的Bug。

