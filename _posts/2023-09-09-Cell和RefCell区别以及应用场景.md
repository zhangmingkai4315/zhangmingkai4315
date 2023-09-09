---
author: 01
title: Cell和RefCell区别以及应用场景
date: 2023-09-09 17:20:11 +0800
categories: [Rust编程, 内存管理]
tags: [rust]     # TAG names should always be lowercase
---

Rust为了确保内存访问的安全，对于访问对象T, 只能允许多个不可变的对象（&T）存在，或者一个可变对象(&mut T)的存在。这个限制条件会在编译器编译期间进行安全检查， 如果出现了违反规则的情况，则编译失败。程序无法执行。 

但是这个条件因为太过于严格，会导致很多程序编写上的问题，因此Rust也提供了一种特例，允许通过程序自身来控制可变和不可变的切换。Rust提供了针对单线程的版本和多线程的版本的数据结构来实现：

- Cell, RefCell和 OnceCell 
- Mutex, RwLock和 OnceLock  

这些类型可以允许在引用情况下，进行对象的修改 也就是 通过内部可变性的特性对于 &T 来修改对象 。但是不同的类型又存在差异 ： 

- ~~Cell<T>  实现内部可变性 需要类型T 具备 Copy 约束， 而RefCell则可以接受非Copy的类型数据。 ~~
- Cell 不能获取内部对象的可变引用，仅可以设置内部的值，和其他类型交换内部的值和复制内部值， 这些都是通过对应的Move或者Copy语义实现。 
- Cell本身类似于&mut T ，没有额外的封装开销， 而且操作都是基于Copy或者Move提供的是值，不会让你获取到一个内部存储数据的指针， 而RefCell则提供的是引用类型, 可以获取可变和不可变的引用类型
- Cell 本身不会进行运行时检查，RefCell的借用检查基于计数实现，是在运行期进行，在实际修改的时候会检查是否之前没有被借用过。
- RefCell的使用更灵活，但是由于本身可能会导致panic 一般来说 如果Cell能够满足条件，有限使用Cell  

Cell 允许使用take来获取对应的内部数据，但是获取后，原来位置上的数据使用Default.default()代替， 而get则使用Copy语义获取对应的值 。 

```rust
use std::cell::Cell;

struct O {
    r: u8,
    s: Cell<u8>,
}

let m = O{
    r: 0,
    s: Cell::new(0),
};

let m_value = 100;
m.s.set(10);
assert_eq!(m.s.take(), 10);
assert_eq!(m.s.take(), 0);
m.s.set(10);
assert_eq!(m.s.get(), 10);
```

RefCell 通过下面的两种方式获取对应的不可变借用类型，和可变借用类型。 但是这种方式要注意是否可能会存在对应数据借用冲突，否则程序会可能出现panic 。 如果可能会存在冲突，使用try_borrow和try_borrow_mut来获取一个Result类型 

```rust
pub fn borrow(&self) -> Ref<T>
pub fn borrow_mut(&self) -> RefMut<T>
```

```rust
let shared_map = Rc::new(RefCell::new(HashMap::new()));
{
    let mut map: RefMut<'_, _ > = shared_map.borrow_mut();
    map.insert("h", 1);
    map.insert("a",2);
}

let total: i32= shared_map.borrow().values().sum();
println!("{total}")
    
```

另外一个实例： 

```rust
#[derive(Default, Debug)]
struct Node{
    v: i64,
    children: Vec<Rc<RefCell<Node>>>
}

impl Node {
    fn new(val: i64) -> Rc<RefCell<Node>>{
        Rc::new(RefCell::new(Node{
            v: val,
            ..Node::default()
        }))
    }

    fn sum(&self) -> i64 {
        self.v + self.children.iter().map(|c| c.borrow().sum()).sum::<i64>()
    }
}

let root = Node::new(1);
root.borrow_mut().children.push(Node::new(2));
let subtree = Node::new(3);
subtree.borrow_mut().children.push(Node::new(4));
root.borrow_mut().children.push(subtree);

println!("graph: {root:#?}");
```