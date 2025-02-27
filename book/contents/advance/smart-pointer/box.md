# `Box<T>`堆对象分配
关于作者帅不帅，估计争议还挺多的，但是如果说`Box<T>`是不是Rust中最常见的智能指针，那估计没有任何争议。因为`Box<T>`允许你将一个值分配到堆上，然后在栈上保留一个智能指针指向堆上的数据。

之前我们在[所有权章节](https://course.rs/basic/ownership/ownership.html#栈stack与堆heap)简单讲过堆栈的概念，这里再补充一些。

## Rust中的堆栈
高级语言Python/Java等往往会弱化堆栈的概念，但是要用好C/C++/Rust，就必须对堆栈有深入的了解，原因是两者的内存管理方式不同: 前者有GC垃圾回收机制， 因此无需你去关心内存的细节。

栈内存从高位地址向下增长，且栈内存是连续分配的，一般来说**操作系统对栈内存的大小都有限制**，因此C语言中无法创建任意长度的数组。在Rust中, `main`线程的[栈大小是`8MB`](https://zhuanlan.zhihu.com/p/446039229)，普通线程是`2MB`，在函数调用时会在其中创建一个临时栈空间，调用结束后Rust会让这个栈空间里的对象自动进入`Drop`流程，最后栈顶指针自动移动到上一个调用栈顶，无需程序员手动干预，因而栈内存申请和释放是非常高效的。

与栈相反，堆上内存则是从低位地址向上增长，**堆内存通常只受物理内存限制**，而且通常是不连续的, 因此从性能的角度看，栈往往比对堆更高。

相比其它语言，Rust堆上对象还有一个特殊之处，它们都拥有一个所有者，因此受所有权规则的限制：当赋值时，发生的是所有权的转移(只需浅拷贝栈上的引用或智能指针即可)， 例如以下代码：
```rust
fn main() {
    let b = foo("world");
    println!("{}", b);
}

fn foo(x: &str) -> String {
    let a = "Hello, ".to_string() + x;
    a
}
```

在`foo`函数中，`a`是`String`类型，它其实是一个智能指针结构体，该智能指针存储在函数栈中，指向堆上的字符串数据。当被从`foo`函数转移给`main`中的`b`变量时，栈上的智能指针被复制一份赋予给`b`，而底层数据无需发生改变，这样就完成了所有权从`foo`函数内部到`b`的转移.

#### 堆栈的性能
很多人可能会觉得栈的性能肯定比堆高，其实未必。 由于我们在后面的性能专题会专门讲解堆栈的性能问题，因此这里就大概给出结论:

- 小型数据，在栈上的分配性能和读取性能都要比堆上高
- 中型数据，栈上分配性能高，但是读取性能和堆上并无区别，因为无法利用寄存器或CPU高速缓存，最终还是要经过一次内存寻址
- 大型数据，只建议在堆上分配和使用

总之栈的分配速度肯定比堆上快，但是读取速度往往取决于你的数据能不能放入寄存器或CPU高速缓存。 因此不要仅仅因为堆上性能不如栈这个印象，就总是优先选择栈，导致代码更复杂的实现。

## Box的使用场景
由于`Box`是简单的封装，除了将值存储在堆上外，并没有其它性能上的损耗。而性能和功能往往是鱼和熊掌，因此`Box`相比其它智能指针，功能较为单一，可以在以下场景中使用它:

- 特意的将数据分配在堆上
- 数据较大时，又不想在转移所有权时进行数据拷贝
- 类型的大小在编译期无法确定，但是我们又需要固定大小的类型时
- 特征对象，用于说明对象实现了一个特征，而不是某个特定的类型

以上场景，我们在本章将一一讲解，后面车速较快，请系好安全带。

#### 使用`Box<T>`将数据存储在堆上
如果一个变量拥有一个数值`let a = 3`, 那变量`a`必然是存储在栈上的，那如果我们想要`a`的值存储在堆上就需要使用`Boxt<T>`: 
```rust
fn main() {
    let a = Box::new(3);
    println!("a = {}", a); // a = 3
    
    // 下面一行代码将报错
    // let b = a + 1; // cannot add `{integer}` to `Box<{integer}>`
}
```

这样就可以创建一个智能指针指向了存储在堆上的`5`，并且`a`持有了该指针。在本章的引言中，我们提到了智能指针往往都实现了`Deref`和`Drop`特征，因此：

- `println!`可以正常打印出`a`的值，是因为它隐式的调用了`Deref`对智能指针`a`进行了解引用
- 最后一行代码`let b = a + 1`报错，是因为在表达式中，我们无法自动隐式的执行`Deref`解引用操作, 你需要使用`*`操作符`let b = *a + 1`，来显式的进行解引用
- `a`持有的智能指针将在作用结束(`main`函数结束)时，被释放掉，这是因为`Box<T>`实现了`Drop`特征

以上的例子在实际代码中其实很少会存在，因为将一个简单的值分配到堆上并没有太大的意义。将其分配在栈上，由于寄存器、CPU缓存的原因，它的性能将更好，而且代码可读性也更好。

#### 避免栈上数据的拷贝
当栈上数据转移所有权时，实际上是把数据拷贝了一份，最终新旧变量各自拥有不同的数据，因此所有权并未转移。

而堆上则不然，底层数据并不会被拷贝，转移所有权仅仅是复制一份栈中的指针，再将新的指针赋予新的变量，然后让拥有旧指针的变量失效，最终完成了所有权的转移:
```rust
fn main() {
    // 在栈上创建一个长度为1000的数组
    let arr = [0;1000];
    // 将arr所有权转移arr1，由于`arr`分配在栈上，因此这里实际上是直接重新深拷贝了一份数据
    let arr1 = arr;

    // arr和arr1都拥有各自的栈上数组，因此不会报错
    println!("{:?}",arr.len());
    println!("{:?}",arr1.len());

    // 在堆上创建一个长度为1000的数组，然后使用一个智能指针指向它
    let arr = Box::new([0;1000]);
    // 将堆上数组的所有权转移给arr1, 由于数据在堆上，因此仅仅拷贝了智能指针的结构体，底层数据并没有被拷贝
    // 所有权顺利转移给arr1，arr不再拥有所有权
    let arr1 = arr;
    println!("{:?}",arr1.len());
    // 由于arr不再拥有底层数组的所有权，因此下面代码将报错
    // println!("{:?}",arr.len());
}
```

从以上代码，可以清晰看出大块的数据为何应该放入堆中，此时`Box`就成为了我们最好的帮手.

#### 将动态大小类型变为Sized固定大小类型
Rust需要在编译时知道类型占用多少空间, 如果一种类型在编译时无法知道具体的大小，那么被称为动态大小类型DST。

其中一种无法在编译时知道大小的类型是**递归类型**：在类型定义中又使用到了自身，或者说该类型的值的一部分可以是相同类型的其它值，这种值的嵌套理论上可以无限进行下去，所以Rust不知道递归类型需要多少空间:
```rust
enum List {
    Cons(i32, List),
    Nil,
}
```

以上就是函数式语言中常见的`Cons List`，它的每个节点包含一个`i32`值，还包含了一个新的`List`，因此这种嵌套可以无限进行下去，然后Rust认为该类型是一个DST类型，并给予报错:
```console
error[E0072]: recursive type `List` has infinite size //递归类型`List`拥有无限长的大小
 --> src/main.rs:3:1
  |
3 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
4 |     Cons(i32, List),
  |               ---- recursive without indirection
```

此时若想解决这个问题，就可以使用我们的`Boxt<T>`:
```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

只需要将`List`存储到堆上，然后使用一个智能指针指向它，即可完成从DST到Sized类型(固定大小类型)的华丽转变.

#### 特征对象
在Rust中，想实现不同类型组成的数组只有两个办法：枚举和特征对象，前者限制较多，因此后者往往是最常用的解决办法。

```rust
trait Draw {
    fn draw(&self);
}

struct Button {
    id: u32
}
impl Draw for Button {
    fn draw(&self) {
        println!("这是屏幕上第{}号按钮",self.id)
    }
}

struct Select {
    id: u32
}

impl Draw for Select {
    fn draw(&self) {
        println!("这个选择框贼难用{}",self.id)
    }
}

fn main() {
    let elems: Vec<Box<dyn Draw>> = vec![
        Box::new(Button{id: 1}),
        Box::new(Select{id: 2})
    ];

    for e in elems {
        e.draw()
    }
}
```

以上代码将不同类型的`Button`和`Select`包装成`Draw`特征的特征对象，放入一个数组中，`Box<dyn Draw>`就是特征对象。

其实，特征也是DST类型，而特征对象在做的也是将DST类型转换为固定大小类型。

## Box内存布局
先来看看`Vec<i32>`的内存布局:
```rust
(stack)    (heap)
┌──────┐   ┌───┐
│ vec1 │──→│ 1 │
└──────┘   ├───┤
           │ 2 │
           ├───┤
           │ 3 │
           ├───┤
           │ 4 │
           └───┘
```

之前提到过`Vec`和`String`都是智能指针，从上图可以看出，该智能指针存储在栈中，然后指向堆上的数组数据。

那如果数组中每个元素都是一个`Box`对象呢？来看看`Vec<Box<i32>>`的内存布局:
```rust
(stack)    (heap)   ┌───┐
┌──────┐   ┌───┐ ┌─→│ 1 │
│ vec2 │──→│B1 │─┘  └───┘
└──────┘   ├───┤    ┌───┐
           │B2 │───→│ 2 │
           ├───┤    └───┘
           │B3 │─┐  ┌───┐
           ├───┤ └─→│ 3 │
           │B4 │─┐  └───┘
           └───┘ │  ┌───┐
                 └─→│ 4 │
                    └───┘
```

上面的`B1`代表被`Box`分配到堆上的值`1`。

可以看出智能指针`vec2`依然是存储在栈上，然后指针指向一个堆上的数组，该数组中每个元素都是一个`Box`智能指针，最终`Box`智能指针又指向了存储在堆上的实际值。

因此当我们从数组中取出某个元素时，取到的是对应的智能指针`Box`，需要对该智能指针进行解引用，才能取出最终的值:
```rust
fn main() {
    let arr = vec![Box::new(1), Box::new(2)];
    let (first,second) = (&arr[0],&arr[1]);
    let sum = **first + **second;
}
```

以上代码有几个值得注意的点:

- 使用`&`借用数组中的元素，否则会报所有权错误
- 表达式不能隐式的解引用，因此必须使用`**`做两次解引用，第一次将`&Box<i32>`类型转成`Box<i32>`，第二次将`Box<i32>`转成`i32`


## Box::leak
`Box`中还提供了一个非常有用的关联函数：`Box::leak`，它可以消费掉`Box`并且强制目标值从内存中泄漏，读者可能会觉得，这有啥用啊？

其实还真有点用，例如，你可以把一个`String`类型，变成一个`'static`生命周期的`&str`类型: 
```rust
fn main() {
   let s = gen_static_str();
   println!("{}",s);
}

fn gen_static_str() -> &'static str{
    let mut s = String::new();
    s.push_str("hello, world");

    Box::leak(s.into_boxed_str())
}
```

在之前的代码中，如果`String`创建于函数中，那么返回它的唯一方法就是转移所有权给调用者`fn move_str() -> String`，而通过`Box::leak`我们不仅返回了一个`&str`字符串切片，它还是`'static`类型的！

要知道真正具有`'static`生命周期的往往都是编译期就创建的值，例如`let v = "hello,world"`, 这里`v`是直接打包到二进制可执行文件中的，因此该字符串具有`'static`生命周期，再比如`const`常量。

又有读者要问了，我还可以手动为变量标注`'static`啊。其实你标注的`'static`只是用来忽悠编译器的，但是超出作用域，一样被释放回收。而使用`Box::leak`就可以将一个运行期的值转为`'static`。

#### 使用场景
光看上面的描述，大家可能还是云里雾里、一头雾水。

那么我说一个简单的场景，**你需要一个在运行期初始化的值，但是可以全局有效，也就是和整个程序活得一样久**, 那么久可以使用`Box::leak`，例如有一个存储配置的结构体实例，它是在运行期动态插入内容，那么就可以将其转为全局有效，虽然`Rc/Arc`也可以实现此功能，但是`Box::leak`是性能最高的.

## 总结
`Box`背后式调用`jemalloc`来做内存管理，所以堆上的空间无需我们的手动管理。与此类似，带GC的语言中的对象也是借助于box概念来实现的，一切皆对象 = 一切皆box， 只不过我们无需自己去box罢了。

其实很多时候，编译器的鞭笞可以助我们更快的成长，例如所有权规则里的借用、move、生命周期就是编译器在教我们做人，哦不是，是教我们深刻理解堆栈、内存布局、作用域等你在其它GC语言无需去关注的东西。刚开始是很痛苦，但是一旦熟悉了这套规则，写代码的效率和代码本身的质量将飞速上升，直到你用Java开发的效率写出Java代码不可企及的性能和安全性，最终Rust语言所谓的开发效率低、心智负担高，对你来说终究不是个事。

因此, 不要怪Rust，**它只是在帮我们成为那个更好的程序员，而这些苦难终究成为我们走向优秀的垫脚石**，
