[此文在掘金上的地址](https://juejin.im/post/5e0d4c686fb9a048401cff26)

## 前言

> 想要成为一名`iOS开发高手`，免不了阅读源码。以下是笔者在`OC源码探索`中梳理的一个小系列——**类与对象篇**，欢迎大家阅读指正，同时也希望对大家有所帮助。
>
> 1. [OC源码分析之对象的创建](https://juejin.im/post/5de08bf85188254fc26bc242)
> 2. [OC源码分析之isa](https://juejin.im/post/5e0d4c686fb9a048401cff26)
> 3. 未完待续...

## 1. `isa`介绍

### 1.1 `isa`是什么

在 [OC源码分析之对象的创建](https://juejin.im/post/5de08bf85188254fc26bc242) 一文中，我们知道`alloc`底层会调用`calloc`分配内存，接着就是`initInstanceIsa(cls, hasCxxDtor)`，顾名思义是初始化对象的isa，其关键代码如下

````c
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    assert(!cls->instancesRequireRawIsa());
    assert(hasCxxDtor == cls->hasCxxDtor());

    // 留意这里的true
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        assert(!DisableNonpointerIsa);
        assert(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        assert(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}

````

1. 关于`Tagged Pointer`

据说，为了节省内存和提高执行效率，苹果提出了`Tagged Pointer`的概念。对于 64 位程序，引入`Tagged Pointer`后，相关逻辑能减少一半的内存占用，以及 **3倍** 的访问速度提升，**100倍** 的创建、销毁速度提升。

> `Tagged Pointer`首次应用于`iPhone 5s`设备上，现在几乎都应用`Tagged Pointer`了。想了解更多关于`Tagged Pointer`的内容可戳 [深入理解 Tagged Pointer](https://www.infoq.cn/article/deep-understanding-of-tagged-pointer/)。

2. `isa_t`类型

点击`objc_object::initIsa()`中的`isa`，发现`isa`是`isa_t`类型
````c
struct objc_object {
private:
    isa_t isa;
    
    ... // 一些isa的公有、私有方法
};
````

而`isa_t`实际上是一个`union`（即联合体，也叫共用体）

> 这里先普及一下`struct`和`union`的区别
> 1. 两者都可以包含多个不同类型的数据，如`int`、`double`、`Class`等。
> 2. 在`struct`中各成员有各自的内存空间，一个`struct`变量的内存总长度大于等于各成员内存长度之和；而在`union`中，各成员共享一段内存空间，一个`union`变量的内存总长度等于各成员中内存最长的那个成员的内存长度。
> 3. 对`struct`中的成员进行赋值，不会影响其他成员的值；对`union`中的成员赋值时，每次只能给一个成员赋值，同时其它成员的值也就不存在了。

`isa_t`包含了`cls`和`bits`两个成员变量，其结构如下

````c
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
````

### 1.2 `isa`的`bits`成员变量

1. 位域

这里普及一下位域的概念

> 位域是一种数据结构，可以把数据以位的形式紧凑的储存，并允许程序员对此结构的位进行操作。
> * 优点：
>   * 节省储存空间；
>   * 可以很方便的访问一个整数值的部分内容从而可以简化程序源代码。
> * 缺点：
>   * 其内存分配与内存对齐的实现方式依赖于具体的机器和系统，在不同的平台可能有不同的结果，这导致了位段在本质上是不可移植的。

`isa`的`bits`成员变量类型是`uintptr_t`，它实质上是个`unsigned long`
````c
typedef unsigned long           uintptr_t;
````

在`64位`CPU架构下`bits`长度为64位，也就是8字节，其各个位的存储就使用了位域，即`ISA_BITFIELD`。

2. `ISA_BITFIELD`

接下来看一下`ISA_BITFIELD`的源码（由于笔者是用`macOS`项目研究OC底层源码，所以这里以`x86_64`架构为例）

````c
# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
#   define ISA_BITFIELD                                                        \
      uintptr_t nonpointer        : 1;                                         \
      uintptr_t has_assoc         : 1;                                         \
      uintptr_t has_cxx_dtor      : 1;                                         \
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
      uintptr_t magic             : 6;                                         \
      uintptr_t weakly_referenced : 1;                                         \
      uintptr_t deallocating      : 1;                                         \
      uintptr_t has_sidetable_rc  : 1;                                         \
      uintptr_t extra_rc          : 8
#   define RC_ONE   (1ULL<<56)
#   define RC_HALF  (1ULL<<7)
````

首先明确一点，在`64位`CPU架构下`isa指针`的长度也是`8字节`，它可以存储足够多的内容，苹果为了优化性能，存储类地址只用了一部分位（`x86_64`下是44位，`arm64`下是33位），剩下的位用来存储一些其它信息。

具体分析一下`ISA_BITFIELD`位域各成员的表示意义：
* `nonpointer`：表示是否对 `isa指针` 开启指针优化。
    * 0：不优化，是纯`isa指针`，当访问`isa`指针时，直接返回其成员变量`cls`
    * 1：优化，即`isa 指针`内容不止是类地址，还包含了类的一些信息、对象的引用计数等。
* `has_assoc`：是否有关联对象。
* `has_cxx_dtor`：该对象是否有C++或Objc的析构器。
    * 如果有析构函数，则需要做一些析构的逻辑处理；
    * 如果没有，则可以更快的释放对象。
* `shiftcls`：存储类指针的值。开启指针优化的情况下，在 `x86_64` 架构有 **44位** 用来存储类指针，`arm64` 架构中有 **33位** 。
* `magic`：用于调试器判断当前对象是真的对象，还是一段没有初始化的空间。
* `weakly_referenced`：用于标识对象是否被指向或者曾经被指向一个`ARC`的弱变量，没有弱引用的对象释放的更快。
* `deallocating`：标识对象是否正在释放内存。
* `has_sidetable_rc`：对象的引用计数值是否有进位。
* `extra_rc`：表示该对象的引用计数值。`extra_rc`只是存储了额外的引用计数，实际的引用计数公式：`实际引用计数 = extra_rc + 1`。这里占了8位，所以理论上可以存储的最大引用计数是：`2^8 - 1 + 1 = 256`（`arm64`CPU架构下的`extra_rc`占19位，可存储的最大引用计数为`2^19 - 1 + 1 = 524288`）。
    * 与`has_sidetable_rc`的关联：当对象的最大引用计数超过界限后，`has_sidetable_rc`的值为1，否则为0

### 1.3 `isa`的`cls`成员变量

分析完`isa`的位域，接下来就只剩下`cls`，它是`Class`类型，同样上源码

````c
typedef struct objc_class *Class;
// 顺便了解一下id的类型，显然id是个指针变量，它的值只有一个isa变量
typedef struct objc_object *id;

struct objc_class : objc_object {
    // Class ISA; 
    Class superclass;
    cache_t cache;             
    class_data_bits_t bits;

    class_rw_t *data() { 
        return bits.data();
    }
    ... // 一些方法
};

struct objc_object {
private:
    isa_t isa;
    
    ... // 一些isa的公有、私有方法
};

````
从源码得知，`Class`实际上是`objc_class`结构体的指针变量，而`objc_class`又继承自`objc_object`，因此`Class`这个结构体指针变量的值内部有一个`isa`成员变量（类型为`isa_t`），这个`isa`成员变量在`64位`CPU架构下是8字节，且排在`objc_class`结构体的前8字节。

### 1.4 `isa`的作用

通过对`isa`的位域说明，我们知道`shiftcls`存储的是类指针的值。在`x86_64`架构下，`shiftcls`占用44位，也就是从第3~46位，将 [3, 46]位 全部填充1，[0, 2]位 和 [47~63]位 都补0，得到0x7ffffffffff8，也就是`ISA_MASK`的值。故，`isa & ISA_MASK`会得到`shiftcls`存储的类指针的值。这也就是所谓`MASK`的作用。

如下图所示
![](https://user-gold-cdn.xitu.io/2020/1/26/16fe04a1ad504624?w=796&h=950&f=png&s=733485)

下面用一个例子说明`isa`的作用

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe03cb1b98b31a?w=1596&h=628&f=png&s=107904)

此时通过`lldb`命令调试

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe054af5b993c3?w=1306&h=516&f=png&s=105946)

> 说明：
> 1. `0x001d800100001129`是`对象p`的`isa`值，通过`isa & ISA_MASK`运算得到的`0x0000000100001128`就是`Person`类的地址
> 2. 证明【1】：通过`p/x Person.class`直接打印`Person`类地址，显然得到的是`0x0000000100001128`，如此【1】证明成立！

**结论：`isa`将对象和类关联起来，起到了中间桥梁的作用。**

> 思考：如果不用`ISA_MASK`，那么如何证明`isa`的这个作用呢？——答案将在文末补充。

### 1.5 `isa`的初始化补充

最后补充一下`isa`的初始化。还记得初始化`isa`的入口吗？是`initIsa(cls, true, hasCxxDtor);`，此时`nonpointer`的值是`true`，再看`SUPPORT_INDEXED_ISA`的定义

````c
#if __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
#   define SUPPORT_INDEXED_ISA 1
#else
#   define SUPPORT_INDEXED_ISA 0
#endif
````

在`x86_64`下，`SUPPORT_INDEXED_ISA`是0，所以`isa`的初始化最终会来到
````c
isa_t newisa(0);

// 使用ISA_MAGIC_VALUE(0x001d800000000001ULL)赋值给bits
// nonpointer为1，magic为1d，其他变量为零
newisa.bits = ISA_MAGIC_VALUE;
// hasCxxDtor是从类的isa中取出的
newisa.has_cxx_dtor = hasCxxDtor;
// 将cls右移3位后赋值给shiftcls
newisa.shiftcls = (uintptr_t)cls >> 3;

isa = newisa;
````
> 碍于篇幅，这里不继续深入`hasCxxDtor`。

## 2 `isa`指向图

通过以上的源码分析，我们认识到对象的`isa指针`指向了对象所属的类。而类本身也有一个`isa`指针，它指向的又是什么呢？

苹果官方有个`isa`指向图，即

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe0db0a22ebfba?w=1144&h=1170&f=png&s=240747)

从图可知，类的`isa指针`指向的是类的元类。现在我们来验证一下吧。

### 2.1 准备工作

创建`Teacher`类、`Person`类，其中`Person`类继承于`NSObject`，`Teacher`类继承于`Person`类。

> 对比`isa指向图`，对号入座后就是，`Teacher`类相当于`Subclass`，`Person`类相当于`Superclass`，`NSObject`相当于`Root class`。

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe144184337a58?w=1792&h=974&f=png&s=184522)

### 2.2 验证过程

1. 获取`teacher`对象的类（结果是`Teacher`类，地址为`0x0000000100001230`）
````c
(lldb) x/4gx teacher
0x100f59400: 0x001d800100001231 0x0000000000000000
0x100f59410: 0x636f72504b575b2d 0x70756f7247737365
(lldb) p/x 0x001d800100001231 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $3 = 0x0000000100001230      // 对象teacher的类地址
(lldb) po $3
Teacher     // 对象teacher的类
````

2. 获取`Teacher`类的元类（结果是`Teacher元类`，地址为`0x0000000100001208`）
````c
(lldb) x/4gx Teacher.class
0x100001230: 0x001d800100001209 0x00000001000011e0
0x100001240: 0x0000000100f61150 0x0000000100000003
(lldb) p/x 0x001d800100001209 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $5 = 0x0000000100001208      // Teacher类的元类地址
(lldb) po $5
Teacher     // Teacher类的元类
````

```!
Teacher类 和 Teacher元类 地址不一样
```

3. 获取`person`对象的类（结果是`Person`类，地址为`0x00000001000011e0`），以及类的元类（结果是`Person元类`，地址为`0x00000001000011b8`）
````c
(lldb) x/4gx person
0x100f60a30: 0x001d8001000011e1 0x0000000000000000
0x100f60a40: 0x0000000000000002 0x00007fff9b855588
(lldb) p/x 0x001d8001000011e1 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $8 = 0x00000001000011e0      // 对象person的类地址
(lldb) po $8
Person      // 对象person的类

(lldb) x/4gx Person.class
0x1000011e0: 0x001d8001000011b9 0x0000000100b38140
0x1000011f0: 0x0000000100f61030 0x0000000100000003
(lldb) p/x 0x001d8001000011b9 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $10 = 0x00000001000011b8     // Person类的元类地址
(lldb) po $10
Person      // Person类的元类
````

4. 获取`object`对象的类（结果是`NSObject`类，地址为`0x0000000100b38140`），以及类的元类（结果是`NSObject元类`，地址为`0x0000000100b380f0`）
````c
(lldb) x/4gx object
0x100f5cc50: 0x001d800100b38141 0x0000000000000000
0x100f5cc60: 0x70736e494b575b2d 0x574b57726f746365
(lldb) p/x 0x001d800100b38141 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $12 = 0x0000000100b38140     // 对象object的类地址
(lldb) po $12   
NSObject    // 对象object的类

(lldb) x/4gx NSObject.class
0x100b38140: 0x001d800100b380f1 0x0000000000000000
0x100b38150: 0x0000000101913060 0x0000000200000003
(lldb) p/x 0x001d800100b380f1 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $14 = 0x0000000100b380f0     // NSObject类的元类地址
(lldb) po $14
NSObject    // NSObject类的元类
````

5. 获取`Teacher元类`的元类，`Person元类`的元类，以及`NSObject元类`的元类
````c
(lldb) x/4gx 0x0000000100001208     //  Teacher元类
0x100001208: 0x001d800100b380f1 0x00000001000011b8
0x100001218: 0x000000010186f950 0x0000000400000007
(lldb) p/x 0x001d800100b380f1 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $16 = 0x0000000100b380f0     // NSObject元类
(lldb) po $16
NSObject    // NSObject元类

(lldb) x/4gx 0x00000001000011b8     // Person元类
0x1000011b8: 0x001d800100b380f1 0x0000000100b380f0
0x1000011c8: 0x0000000101905a50 0x0000000300000007
(lldb) p/x 0x001d800100b380f1 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $17 = 0x0000000100b380f0     // NSObject元类
(lldb) po $17
NSObject    // NSObject元类

(lldb) x/4gx 0x0000000100b380f0     // NSObject元类
0x100b380f0: 0x001d800100b380f1 0x0000000100b38140
0x100b38100: 0x0000000101903820 0x0000000500000007
(lldb) p/x 0x001d800100b380f1 & 0x00007ffffffffff8  // isa & ISA_MASK
(long) $18 = 0x0000000100b380f0     // NSObject元类
(lldb) po $18
NSObject    // NSObject元类
````

### 2.3 `isa`指向结论

基于【2.2】的验证过程，可以得出结论：
* 对象的`isa指针` 指向 对象的所属类（如`person`对象的`isa`指向`Person`类）
* 类的`isa指针` 指向 类的元类（如`Person`类的`isa`指向`Person元类`）
* 元类的`isa指针` 指向 根元类（如`Person元类`的`isa`指向`NSObject元类`）
    * `NSObject类`的元类是`根元类`
    * `NSObject元类`的`isa指针` 指向自身（是个圆圈）

> 思考：如果Person类继承的是NSProxy，相关isa指向是怎样的呢？感兴趣的可以去试试。

### 2.4 继承关系的证明

类的继承关系证明过程：（以 -> 表示 继承自）
````c
(lldb) p class_getSuperclass(Teacher.class)
(Class) $19 = Person    // Teacher类 -> Person类

(lldb) p class_getSuperclass(Person.class)
(Class) $20 = NSObject  // Person类 -> NSObject类

(lldb) p class_getSuperclass(NSObject.class)
(Class) $21 = nil       // NSObject类 -> nil
````

元类的继承关系证明过程：（以 -> 表示 继承自）
````c
// 0x0000000100001208 是 Teacher元类
(lldb) p/x class_getSuperclass((Class)0x0000000100001208)
(Class) $17 = 0x00000001000011b8    // Person元类
(lldb) po $17
Person      // Teacher元类 -> Person元类

// 0x00000001000011b8 是 Person元类
(lldb) p/x class_getSuperclass((Class)0x00000001000011b8)
(Class) $22 = 0x0000000100b380f0    // NSObject元类（根元类）
(lldb) po $22
NSObject    // Person元类 -> 根元类

// 0x0000000100b380f0 是 根元类
(lldb) p/x class_getSuperclass((Class)0x0000000100b380f0)
(Class) $23 = 0x0000000100b38140 NSObject   // NSObject类（根类）
(lldb) po $23
NSObject    // 根元类 -> 根类
````

```!
根元类继承自根类（NSObject元类 -> NSObject类），根类继承自nil（NSObject类 -> nil）
```

### 2.5 `isa`指向图涂鸦版

把上面的例子涂在`isa`指向图上，就得到了下图

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe1d5a860ed423?w=1326&h=1364&f=png&s=365981)


## 3. 总结

1. `isa`是`isa_t`结构，采用 **联合体+位域** 的搭配来设计：在不同的位上显示不同的内容，以此来节省储存空间，进而优化内存。
2. `isa`包含了`cls`和`bits`两个成员变量，这两个成员变量在`64位`CPU架构下的长度都是8字节，所以`isa`在`64位`CPU架构下的长度也是8字节。
3. `isa`的位域上存储了一些对象与类的信息，并将对象与类关联起来，起到中间桥梁的作用。
4. `isa`指向图相关结论：
    * 对象的`isa指针` 指向 对象的所属类（如`person`对象的`isa`指向`Person`类）
    * 类的`isa指针` 指向 类的元类（如`Person`类的`isa`指向`Person元类`）
    * 元类的`isa指针` 指向 根元类（如`Person元类`的`isa`指向`NSObject元类`）
        * `根元类`的`isa指针` 指向自身（是个圆圈）
    * 元类的继承关系向上传递（如`Teacher元类` 继承自 `Person元类`）
        * `根元类` 继承自 `根类`
        * `根类` 继承自 `nil`


## 4. 补充

### 4.1 `isa`的作用的证明2

Q：如果不用`ISA_MASK`，那么如何证明`isa`关联了对象和类的作用呢？

A：具体思路是，`shiftcls`在`x86_64`架构下长度是44位，存储在`isa`的 [3, 46]位上，所以可以通过将`isa`的 [0, 2]位、[47, 63]位清零，同样能得到`shiftcls`的值，进而确定类。

如图所示，经过对`isa`的一番运算，成功得到与`Person`类相同的地址。

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe065b0df62b9f?w=1550&h=596&f=png&s=145353)

### 4.2 `NSProxy`的`isa`指向

Q：如果Person类继承的是NSProxy，相关isa指向是怎样的呢？

A：跟`NSObject`一样，两者都是`根类`。
