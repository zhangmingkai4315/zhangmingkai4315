---
author: 01
title: Rust中?Sized使用方式和场景应用
date: 2023-09-09 10:20:11 +0800
categories: [Rust编程, DST]
tags: [rust]     # TAG names should always be lowercase
---

首先 大部分的类型是编译期间就能确定类型的长度的 比如i32 i64 这种，即便是复合类型 Struct如果内部的都是明确的固定类型的话，本身也是可以确定长度的。 因此这些都是Sized .  但是在Rust代码中还有一类是无法明确的在编译期间知道其长度，这类称之为DST(dynamic sized type) , 比如 trait对象 和slices数组 。

对于指向DST类型的指针，本身需要两倍的存储空间： 

- 指向slice数组的 也需要存储额外的slice的数量信息
- 指向trait的 也需要存储一个指向vtable的指针

```rust
use std::mem::size_of;
use std::boxed::Box;
use std::string::ToString;

#[test]
fn test_sized_type(){
    let single_pointer_size = size_of::<&()>();
    let double_pointer_size = 2 * single_pointer_size;

    assert_eq!(double_pointer_size, size_of::<&str>());
    assert_eq!(double_pointer_size, size_of::<&[i32]>());
    assert_eq!(double_pointer_size, size_of::<Box<dyn ToString>>());
    assert_eq!(double_pointer_size, size_of::<&dyn ToString>());
}
```

如果传入参数的时候需要上述的DST类型，可以使用？Sized 标示出来， 不标识的话默认所有的都是Sized。
 默认情况下 泛型类型都是约束Sized类型，因此必须传递编译期间已知类型。同时Rust中大部分的trait都是作为约束来限制传输的参数类型的， 但是？Sized则扩大了范围，使用这种作为trait 约束，可以传递几乎任何类型的对象。


比如下面的例子中： 

- 定义一个FooSized的结构体，内部仅存储一个引用类型
- 定义一个trait Print 实现类型的打印
- 定义FooSized来实现trait Print 

```rust
  struct FooSized<'a, T>(&'a T) where T: 'a;

    trait Print{
        fn print(&self);
    }

    impl<'a, T> Print for FooSized<'a, T> where T: 'a + fmt::Display {
        fn print(&self) {
            println!("{}", self.0)
        }
    }

    #[test]
    fn test_un_sized_type(){
        let h = FooSized("hello");
        h.print();
    }
```

当我们尝试执行print的时候发现会报错，尽管我们传递的是一个引用，但是这个引用是一个&‘staic str， 非固定大小的类型， 因此我们必须使用提供DST的类型约束。 

![image.png](https://cdn.nlark.com/yuque/0/2023/png/21697776/1693657626081-dd3d85ec-23d3-4906-abb0-4b73a80b2804.png#averageHue=%23252830&clientId=u7a5b4f47-a72c-4&from=paste&height=319&id=u8bbae831&originHeight=351&originWidth=1038&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53247&status=done&style=none&taskId=u2ba4ee4f-cfc0-4d75-b958-7460b381fbf&title=&width=943.6363431835968)

修改上述的约束条件： 

```rust
    struct FooSized<'a, T: ?Sized>(&'a T) where T: 'a;

    trait Print{
        fn print(&self);
    }

    impl<'a, T:?Sized> Print for FooSized<'a, T> where T: 'a + fmt::Display {
        fn print(&self) {
            println!("{}", self.0)
        }
    }
```