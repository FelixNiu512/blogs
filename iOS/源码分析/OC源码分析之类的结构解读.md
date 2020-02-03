## 前言

> 想要成为一名`iOS开发高手`，免不了阅读源码。以下是笔者在`OC源码探索`中梳理的一个小系列——**类与对象篇**，欢迎大家阅读指正，同时也希望对大家有所帮助。
>
> 1. [OC源码分析之对象的创建](https://juejin.im/post/5de08bf85188254fc26bc242)
> 2. [OC源码分析之isa](https://juejin.im/post/5e0d4c686fb9a048401cff26)
> 3. [OC源码分析之类的结构解读](https://juejin.im/post/5e2c018a5188254dc42da951)
> 4. 未完待续...

## 1. 类的结构

如果你使用过`Objective-C`（简称`OC`）这门语言开发过应用程序，你一定对`NSObject`不陌生。`OC`里面有两个`NSObject`，一个是我们熟知的`NSObject`类，另一个是`NSObject`协议。协议类似于其他面向对象语言（如`Java`、`C++`）的接口，`NSObject`协议里面定义了一些属性和方法，但本身并未实现，而`NSObject`类遵循了`NSObject`协议，所以`NSObject`类实现了这些方法。

我们跟`NSObject`类打了那么多交道，却不一定对它了如指掌，今天笔者将带大家对`NSObject`类的结构进行全方位的解读。

注意：
> * 本文笔者用的所有源码都是基于苹果开源的`objc4-756.2源码`，文末会附上`github`的地址。
> * 本文采用的是`x86_64`CPU架构，跟`arm64`差别不大，有区别的地方会注明的。

### 1.1 解读类的本质

从`NSObject`类的定义开始

````c
OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0)
OBJC_ROOT_CLASS
OBJC_EXPORT
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}
````

> 可以看到`NSObject`有个`Class`类型的`isa`成员变量，这里大家留意一下。

接下来用`Clang`编译`main.m`，输出`.cpp`文件，看一下`NSObject`类的底层定义

````c
clang -rewrite-objc main.m -o main.cpp
````

打开`main.cpp`文件，找到了`NSObject`

````c
#ifndef _REWRITER_typedef_NSObject
#define _REWRITER_typedef_NSObject
typedef struct objc_object NSObject;
typedef struct {} _objc_exc_NSObject;
#endif

struct NSObject_IMPL {
	Class isa;
};
````

发现`NSObject`类本质上是`objc_object`结构体，同时有定义一个`NSObject_IMPL`结构体（`IMPL`是`implementation`的缩写），里面有`NSObject类`的`isa`成员变量（对应于`OC`时的`NSObject`类定义中的`isa`成员变量）。

此时，笔者特别好奇我们自己定义的类经`Clang`编译后是什么样子，索性看一下吧

````c
@interface Person : NSObject

@property (nonatomic) NSInteger age;

- (void)run;

@end

@implementation Person

- (void)run {
    NSLog(@"I am running.");
}

@end
````

一个简单的`Person`类，有个`age`属性和`run`方法，编译后就是

````c
#ifndef _REWRITER_typedef_Person
#define _REWRITER_typedef_Person
typedef struct objc_object Person;
typedef struct {} _objc_exc_Person;
#endif

extern "C" unsigned long OBJC_IVAR_$_Person$_age;
struct Person_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	NSInteger _age;
};

static void _I_Person_run(Person * self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_mc_9fhhprrj4k92vxzqm3g127z40000gn_T_main_09fc70_mi_0);
}

static NSInteger _I_Person_age(Person * self, SEL _cmd) { return (*(NSInteger *)((char *)self + OBJC_IVAR_$_Person$_age)); }
static void _I_Person_setAge_(Person * self, SEL _cmd, NSInteger age) { (*(NSInteger *)((char *)self + OBJC_IVAR_$_Person$_age)) = age; }
````

可见，`Person`类本质同样是`objc_object`结构体类型，唯一能体现与`NSObject`类之间的继承关系的就是，`Person_IMPL`结构体内部多了个`struct NSObject_IMPL`类型的`NSObject_IVARS`成员变量——即以`NSObject`为根类的继承体系里的所有类，都有个`Class`类型的`isa`成员变量。

### 1.2 `objc_object`结构

如果你对`isa`有所了解，或者有读过笔者的 [OC源码分析之isa](https://juejin.im/post/5e0d4c686fb9a048401cff26) 这篇文章，相信一定对`objc_object`有印象。

这里就直接上源码

````c
struct objc_object {
private:
    isa_t isa;
    
    ... // 一些函数
};

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

* 关于`isa_t`的分析请戳 [OC源码分析之isa](https://juejin.im/post/5e0d4c686fb9a048401cff26)
，里面已经很详细了，这里就不多作赘述。
* `objc_object`结构体内部的方法有五十个左右，大致可分为以下几类
    * 一些关于`isa`的函数，如`initIsa()`、`getIsa()`、`changeIsa()`等
    * 一些弱引用的函数，如`isWeaklyReferenced()`、`setWeaklyReferenced_nolock()`等
    * 一些内存管理函数，如`retain()`、`release()`、`autorelease()`等
    * 两个关联对象函数，分别是`hasAssociatedObjects()`和`setHasAssociatedObjects`

### 1.3 `Class`结构简介

同样先上源码

````c
typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    ... // 一些函数
};
````

从源码可以看出，`Class`是`objc_class`结构体类型的指针变量，继承自`objc_object`结构体。也就是说，`Class`有4个成员变量，且它们在内存存储上是有序的，依次分别是：
1. `isa`：类型是`isa_t`，`64位`下长度为8字节，由于上篇博文已做过分析，这里略过；
2. `superclass`：类型是`Class`，表示继承关系，指向当前类的父类，同样8字节；
3. `cache`：类型是`cache_t`，表示缓存，用于缓存指针和 `vtable`，加速方法的调用。其具体结构如下

````c
struct cache_t {
    struct bucket_t *_buckets;  // 64位下是8字节
    mask_t _mask;               // 64位下是4字节
    mask_t _occupied;           // 64位下是4字节

    ... // 一些函数
};

#if __LP64__
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
#else
typedef uint16_t mask_t;
#endif

typedef unsigned int uint32_t;
````

可见，`cache`这个成员变量长度是16字节。

```!
cache比较重要，关于它的分析笔者将另起一篇博文，这里暂时搁置。
```

4. `bits`：类型是`class_data_bits_t`，用于存储类的数据（类的方法、属性、遵循的协议等信息），其结构如下

````c
struct class_data_bits_t {

    // Values are the FAST_ flags above.
    uintptr_t bits;     // unsigned long

    ... // 一些函数
};
````

其长度也是8字节。根据`bits`成员变量在`objc_object`结构体中的描述，它实质上是`class_rw_t *`加上自定义`rr/alloc`标志，也就是说，最重要的是`class_rw_t`——笔者接下来将重点介绍它。

## 2. `class_rw_t` & `class_ro_t`分析

**`OC`类中的属性、方法还有遵循的协议等信息都保存在`class_rw_t`中**，首先看看`class_rw_t`的结构：
````c
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;
    
    method_array_t methods;         // 方法列表
    property_array_t properties;    // 属性列表
    protocol_array_t protocols;     // 协议列表

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;

#if SUPPORT_INDEXED_ISA
    uint32_t index;
#endif
    ...// 一些函数
};

#if __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
#   define SUPPORT_INDEXED_ISA 1
#else
#   define SUPPORT_INDEXED_ISA 0
#endif
````

发现`class_rw_t`中还有一个被`const`修饰的指针变量 `ro`，是`class_ro_t`结构体指针，其中存储了**当前类在编译期确定的方法、成员变量、属性以及遵循的协议等信息**。

````c
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;     // 方法列表
    protocol_list_t * baseProtocols;    // 协议列表
    const ivar_list_t * ivars;          // 成员变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;    // 属性列表

    // This field exists only when RO_HAS_SWIFT_INITIALIZER is set.
    _objc_swiftMetadataInitializer __ptrauth_objc_method_list_imp _swiftMetadataInitializer_NEVER_USE[0];

    ... // 一些函数
};
````

说明：
> 其实，`rw`是`readwrite`的意思，而`ro`则是`readonly`。

### 2.1 获取`class_rw_t`

要想获取`class_rw_t`指针地址，需要知道`objc_class`的`bits`指针地址，通过对`objc_class`的结构分析得知，`bits`指针地址是`objc_class`首地址偏移32个字节（`isa` + `superclass` + `cache` = 32字节）

也可以从源码得知如何拿到`class_rw_t`指针
````c
// objc_class结构体中
class_rw_t *data() { 
    return bits.data();     // bits是class_data_bits_t类型
}

// class_data_bits_t结构体中
class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
}

// 64位下
#define FAST_DATA_MASK          0x00007ffffffffff8UL
````

> 在`64位下`，`class_rw_t`指针地址是在[3, 46]数据段，所以也可以用`bits & FAST_DATA_MASK`计算出`class_rw_t`指针地址。

接着笔者将通过一个例子来验证`class_rw_t`和`class_ro_t`是否存储了类的信息

### 2.2 准备工作

给`Person`类添加**属性、方法和协议**，代码如下
````c
@protocol PersonProtocol <NSObject>

- (void)walk;

@end

@interface Person : NSObject <PersonProtocol> {
    NSInteger _gender;
}

@property (nonatomic) NSString *name;
@property (nonatomic) NSInteger age;

+ (void)printMyClassName;
- (void)run;

@end

@implementation Person

+ (void)printMyClassName {
    NSLog(@"my class name is Person");
}

- (void)run {
    NSLog(@"I am running.");
}

- (void)walk {
    NSLog(@"I am walking.");
}

@end
````

然后在合适的位置打上断点

![](https://user-gold-cdn.xitu.io/2020/1/31/16ffae6efe474c71?w=1128&h=288&f=png&s=54460)

好了，准备工作完成，下面开始验证

### 2.3 `class_rw_t`验证过程

1. 打印`Person`类
````c
(lldb) x/5gx pcls
0x100002820: 0x001d8001000027f9 0x0000000100b39140
0x100002830: 0x00000001003dc250 0x0000000000000000
0x100002840: 0x0000000102237404
````

说明：
> * `Person`类首地址是`0x100002820`，因此，`0x100002840`是其`bits`地址（32字节就是`0x20`，`0x100002840` = `0x100002820` + `0x20`），`bits`内容是`0x0000000102237404`
> * `0x001d8001000027f9`是`Person`类的`isa`地址，指向`Person元类`
> * `0x0000000100b39140`是`Person`类的`superclass`地址，也就是`NSObject`类首地址
> * `0x00000001003dc250 0x0000000000000000`则是`Person`类的`cache`段

2. 打印`class_rw_t`

````c
// bits & FAST_DATA_MASK
(lldb) p (class_rw_t *)(0x0000000102237404 & 0x00007ffffffffff8)
(class_rw_t *) $1 = 0x0000000102237400
(lldb) p *$1
(class_rw_t) $2 = {
  flags = 2148139008
  version = 0
  ro = 0x0000000100002788
  methods = {
    list_array_tt<method_t, method_list_t> = {
       = {
        list = 0x0000000100002608
        arrayAndFlag = 4294977032
      }
    }
  }
  properties = {
    list_array_tt<property_t, property_list_t> = {
       = {
        list = 0x0000000100002720
        arrayAndFlag = 4294977312
      }
    }
  }
  protocols = {
    list_array_tt<unsigned long, protocol_list_t> = {
       = {
        list = 0x00000001000025a8
        arrayAndFlag = 4294976936
      }
    }
  }
  firstSubclass = nil
  nextSiblingClass = NSUUID
  demangledName = 0x0000000000000000
}
````

这里请大家留意一下`class_rw_t`的几个关键成员变量：
> * `ro`地址是`0x0000000100002788`
> * `methods`的`list`地址是`0x0000000100002608`
> * `properties`的`list`地址是`0x0000000100002720`
> * `protocols`的`list`地址是`0x0000000100002608`

3. 验证`methods`

目前来看，`Person`类至少有6个实例方法，分别是`run`、`walk`以及`name`和`age`的`getter`、`setter`，还有1个类方法，即`printMyClassName`，总计7个方法。

````c
(lldb) p (method_list_t *)0x0000000100002608    // rw的methods的list地址
(method_list_t *) $7 = 0x0000000100002608
(lldb) p *$7
(method_list_t) $8 = {
  entsize_list_tt<method_t, method_list_t, 3> = {
    entsizeAndFlags = 26
    count = 7
    first = {
      name = "walk"
      types = 0x0000000100001e96 "v16@0:8"
      imp = 0x0000000100001530 (CCTest`-[Person walk] at main.m:45)
    }
  }
}
````

正好是7个方法，让我们看看都是哪些（由于`method_list_t`继承自`entsize_list_tt`，可以通过`entsize_list_tt`的`get()`函数一一打印）
````c
(lldb) p $8.get(0)
(method_t) $9 = {
  name = "walk"
  types = 0x0000000100001e96 "v16@0:8"
  imp = 0x0000000100001530 (CCTest`-[Person walk] at main.m:45)
}
(lldb) p $8.get(1)
(method_t) $10 = {
  name = ".cxx_destruct"
  types = 0x0000000100001e96 "v16@0:8"
  imp = 0x0000000100001600 (CCTest`-[Person .cxx_destruct] at main.m:35)
}
(lldb) p $8.get(2)
(method_t) $11 = {
  name = "name"
  types = 0x0000000100001eb1 "@16@0:8"
  imp = 0x0000000100001560 (CCTest`-[Person name] at main.m:27)
}
(lldb) p $8.get(3)
(method_t) $12 = {
  name = "setName:"
  types = 0x0000000100001f4b "v24@0:8@16"
  imp = 0x0000000100001580 (CCTest`-[Person setName:] at main.m:27)
}
(lldb) p $8.get(4)
(method_t) $13 = {
  name = "age"
  types = 0x0000000100001f56 "q16@0:8"
  imp = 0x00000001000015c0 (CCTest`-[Person age] at main.m:28)
}
(lldb) p $8.get(5)
(method_t) $14 = {
  name = "run"
  types = 0x0000000100001e96 "v16@0:8"
  imp = 0x0000000100001500 (CCTest`-[Person run] at main.m:41)
}
(lldb) p $8.get(6)
(method_t) $15 = {
  name = "setAge:"
  types = 0x0000000100001f5e "v24@0:8q16"
  imp = 0x00000001000015e0 (CCTest`-[Person setAge:] at main.m:28)
}
````
显然，`class_rw_t`的`methods`确实包含了`Person`类的全部实例方法，只是多了个`.cxx_destruct`方法。`.cxx_destruct`方法原本是为了`C++`对象析构的，`ARC`借用了这个方法插入代码实现了自动内存释放的工作，关于其原理这里略过不提。

> 思考：类方法`printMyClassName`哪里去了？

4. 验证`properties`

同理，`Person`类至少有`name`和`age`这两个属性，且看

````c
(lldb) p (property_list_t *)0x0000000100002720  // rw的properties的list地址
(property_list_t *) $18 = 0x0000000100002720
(lldb) p *$18
(property_list_t) $19 = {
  entsize_list_tt<property_t, property_list_t, 0> = {
    entsizeAndFlags = 16
    count = 6
    first = (name = "name", attributes = "T@\"NSString\",&,N,V_name")
  }
}
(lldb) p $19.get(0)
(property_t) $20 = (name = "name", attributes = "T@\"NSString\",&,N,V_name")
(lldb) p $19.get(1)
(property_t) $21 = (name = "age", attributes = "Tq,N,V_age")
(lldb) p $19.get(2)
(property_t) $22 = (name = "hash", attributes = "TQ,R")
(lldb) p $19.get(3)
(property_t) $23 = (name = "superclass", attributes = "T#,R")
(lldb) p $19.get(4)
(property_t) $24 = (name = "description", attributes = "T@\"NSString\",R,C")
(lldb) p $19.get(5)
(property_t) $25 = (name = "debugDescription", attributes = "T@\"NSString\",R,C")
````
显然`name`和`age`存储在`properties`中。
> 多余的`属性`也不作赘述。

5. 验证`protocols`

在验证之前，先分析一下`protocol_list_t`，这个结构体并不是继承自`entsize_list_tt`的，其结构如下

````c
struct protocol_list_t {
    // count is 64-bit by accident. 
    uintptr_t count;
    protocol_ref_t list[0]; // variable-size

    ... // 一些函数
}
````

注意到`variable-size`这个注释部分（可变大小），仿佛看到了希望
````c
typedef uintptr_t protocol_ref_t;  // protocol_t *, but unremapped

struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;

    const char *demangledName();

    ... // 一些函数
};
````

`protocol_ref_t`虽然未映射成`protocol_t *`，不过应该可以考虑一下强转，实验一下吧（这次是找到`PersonProtocol`协议）

````c
(lldb) p (protocol_list_t *)0x00000001000025a8  // rw的protocols的list地址
(protocol_list_t *) $26 = 0x00000001000025a8
(lldb) p *$26
(protocol_list_t) $27 = (count = 1, list = protocol_ref_t [] @ 0x00007fb5decb30f8)

(lldb) p (protocol_t *)$26->list[0]
(protocol_t *) $32 = 0x00000001000028a8
(lldb) p *$32
(protocol_t) $33 = {
  objc_object = {
    isa = {
      cls = Protocol
      bits = 4306735304
       = {
        nonpointer = 0
        has_assoc = 0
        has_cxx_dtor = 0
        shiftcls = 538341913
        magic = 0
        weakly_referenced = 0
        deallocating = 0
        has_sidetable_rc = 0
        extra_rc = 0
      }
    }
  }
  mangledName = 0x0000000100001d16 "PersonProtocol" // 出现了！！！
  protocols = 0x0000000100002568
  instanceMethods = 0x0000000100002580
  classMethods = 0x0000000000000000
  optionalInstanceMethods = 0x0000000000000000
  optionalClassMethods = 0x0000000000000000
  instanceProperties = 0x0000000000000000
  size = 96
  flags = 0
  _extendedMethodTypes = 0x00000001000025a0
  _demangledName = 0x0000000000000000
  _classProperties = 0x0000000000000000
}
````

功夫不负有心人，最终验证了`class_rw_t`的`protocols`中含有`Person`类所遵循的`PersonProtocol`协议。

到了这里，**`class_rw_t`确实存储了类的实例方法、属性和遵循的协议了。**

### 2.4 `class_ro_t`验证过程

现在就剩下`ro`了

1. 打印`class_ro_t`

````c
(lldb) p $1->ro
(const class_ro_t *) $38 = 0x0000000100002788
(lldb) p *$38
(const class_ro_t) $39 = {
  flags = 388
  instanceStart = 8
  instanceSize = 32
  reserved = 0
  ivarLayout = 0x0000000100001d2e "\x11"
  name = 0x0000000100001d0f "Person"
  baseMethodList = 0x0000000100002608
  baseProtocols = 0x00000001000025a8
  ivars = 0x00000001000026b8
  weakIvarLayout = 0x0000000000000000
  baseProperties = 0x0000000100002720
  _swiftMetadataInitializer_NEVER_USE = {}
}
````

有没有发现什么！**`class_ro_t`的方法、属性和协议的地址都与`class_rw_t`的一致**，既然指向的是同一块内存空间，**显然`class_ro_t`也存储了`Person`类的实例方法、属性和协议**。

与`class_rw_t`不同的是，`class_ro_t`多了一个`ivars`列表，里面存放的应该是`Person`类的成员变量。

2. 验证`ivars`

`Person`类的成员变量有：`_gender`、`_name`和`_age`

所幸`ivar_list_t`是继承自`entsize_list_tt`的，`get()`函数又可以用了。

````c
(lldb) p $39.ivars
(const ivar_list_t *const) $40 = 0x00000001000026b8
(lldb) p *$40
(const ivar_list_t) $41 = {
  entsize_list_tt<ivar_t, ivar_list_t, 0> = {
    entsizeAndFlags = 32
    count = 3
    first = {
      offset = 0x00000001000027e0
      name = 0x0000000100001e83 "_gender"
      type = 0x0000000100001f69 "q"
      alignment_raw = 3
      size = 8
    }
  }
}
(lldb) p $41.get(0)
(ivar_t) $42 = {
  offset = 0x00000001000027e0
  name = 0x0000000100001e83 "_gender"
  type = 0x0000000100001f69 "q"
  alignment_raw = 3
  size = 8
}
(lldb) p $41.get(1)
(ivar_t) $43 = {
  offset = 0x00000001000027e8
  name = 0x0000000100001e8b "_name"
  type = 0x0000000100001f6b "@\"NSString\""
  alignment_raw = 3
  size = 8
}
(lldb) p $41.get(2)
(ivar_t) $44 = {
  offset = 0x00000001000027f0
  name = 0x0000000100001e91 "_age"
  type = 0x0000000100001f69 "q"
  alignment_raw = 3
  size = 8
}
````

完全符合预期，`class_ro_t`确实存储了`Person`类的成员变量。

### 2.5 `rw`和`ro`的联系

为什么`class_rw_t`、`class_ro_t`的方法、属性和协议的地址一致？笔者在`class_data_bits_t`结构体中的`safe_ro()`函数中发现了端倪

````c
const class_ro_t *safe_ro() {
    class_rw_t *maybe_rw = data();
    if (maybe_rw->flags & RW_REALIZED) {
        // maybe_rw is rw
        return maybe_rw->ro;
    } else {
        // maybe_rw is actually ro
        return (class_ro_t *)maybe_rw;
    }
}
````
可见，`rw`不一定是`rw`，也可能是`ro`。实际上，在编译期间，类的`class_data_bits_t *bits`指针指向的是`class_ro_t *`，然后在`OC`运行时调用了`realizeClassWithoutSwift()`（苹果开源的`objc4-756.2源码`是`realizeClassWithoutSwift()`，在此之前的版本是`realizeClass()`方法），这个方法主要做的就是利用编译期确定的`ro`来初始化`rw`：

````c
ro = (const class_ro_t *)cls->data();
if (ro->flags & RO_FUTURE) {
    // This was a future class. rw data is already allocated.
    rw = cls->data();
    ro = cls->data()->ro;
    cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
} else {
    // 一般走这里
    // Normal class. Allocate writeable class data.
    rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);   // 给rw申请内存
    rw->ro = ro;    // 设置rw的ro
    rw->flags = RW_REALIZED|RW_REALIZING;   // 设置flags
    cls->setData(rw);   // 给cls设置正确的rw
}

... // 初始化 rw 的其他字段，更新superclass、meta class

// Attach categories
methodizeClass(cls);
````

在代码的最后，还调用了`methodizeClass()`，其源码如下

````c
static void methodizeClass(Class cls)
{
    runtimeLock.assertLocked();

    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;

    ... // 打印信息

    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }

    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }

    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    if (cls->isRootMetaclass()) {
        addMethod(cls, SEL_initialize, (IMP)&objc_noop_imp, "", NO);
    }

    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);

    ... // 打印信息
    if (cats) free(cats);
    ... // 打印信息
}
````
在这个方法里，**将类自己实现的方法（包括分类）、属性和遵循的协议加载到 `methods`、`properties` 和 `protocols` 列表中。**

这就完美解释了为什么运行时`rw`和`ro`的方法、属性和协议相同。

### 2.6 `rw`和`ro`在运行时的不同之处

目前为止的验证都是基于`Person`类的现有结构，也就是在编译期就确定的，突出不了`class_rw_t`和`class_ro_t`的差异性。接下来笔者会用`runtime`的`api`在运行时为`Person`动态添加一个属性`fly()`方法，再来一试。

1. 添加方法

具体代码如下：
````c
void fly(id obj, SEL sel) {
    NSLog(@"I am flying");
}

class_addMethod([Person class], NSSelectorFromString(@"fly"), (IMP)fly, "v@:");
````

再加一个打印方法，用于打印类的`methods`

````c
void printMethods(Class cls) {
    if (cls == nil) {
        return ;
    }
    CCNSLog(@"------------ print %@ methods ------------", NSStringFromClass(cls));
    uint32_t count;
    Method *methods = class_copyMethodList(cls, &count);
    for (uint32_t i = 0; i < count; i++) {
        Method method = methods[i];
        CCNSLog(@"名字：%@ -- 类型：%s", NSStringFromSelector(method_getName(method)), method_getTypeEncoding(method));
    }
}
````

运行一下看效果，发现添加成功，如图

![](https://user-gold-cdn.xitu.io/2020/2/1/17000387f0c699ff?w=1670&h=1028&f=png&s=320386)

2. 验证过程

先打印`class_rw_t`，即

![](https://user-gold-cdn.xitu.io/2020/2/1/170003ea1b3753fe?w=1770&h=1348&f=png&s=314467)

还有`class_ro_t`

![](https://user-gold-cdn.xitu.io/2020/2/1/170004619eb59e67?w=1540&h=570&f=png&s=196149)

对比后发现，**两者的属性、协议指针地址未发生变化，但是方法的指针地址不一样了**。 由于`class_rw_t`是运行时才初始化的，而`class_ro_t`在编译期间就确定了，因此可以猜测新增的`fly`方法存储在`class_rw_t`的`methods`指针上，`class_ro_t`的`baseMethodList`指针从编译期之后就未发生改变。

下面继续验证，首先看`class_ro_t`的方法列表

![](https://user-gold-cdn.xitu.io/2020/2/1/170004aec2ad5767?w=1626&h=844&f=png&s=241733)

![](https://user-gold-cdn.xitu.io/2020/2/1/170004ccadaa56d4?w=1796&h=954&f=png&s=381069)

OK，编译期就确定的方法都在，并且没有`fly`方法，也就是说`class_ro_t`的方法列表在运行时基本没变。
> `class_ro_t`的属性列表、成员变量列表、协议在运行时都没有发生改变。感兴趣的同学可以自己尝试验证一下。

接着看`class_rw_t`的方法列表

![](https://user-gold-cdn.xitu.io/2020/2/1/1700061f177380d9?w=1698&h=1018&f=png&s=328640)

**`class_rw_t`的`methods`里面数据居然都没有了？** 没办法，这里暂时留个坑吧，笔者也不知道原因。

## 3. 总结

### 3.1 类的结构总结

关于类的结构，我们了解到：

* 类本质上是`objc_object`结构体，也就是类也是对象，即万物是对象。
* 类都包含一个`Class`类型的成员变量`isa`，`Class`是`objc_class`结构体类型的指针变量，内部有4个成员变量，即
    * `isa`：类型是`isa_t`，详细请戳 [OC源码分析之isa](https://juejin.im/post/5e0d4c686fb9a048401cff26)
    * `superclass`：类型是`Class`，表示继承关系，指向类的父类
    * `cache`：类型是`cache_t`，表示缓存，用于缓存指针和 `vtable`，加速方法的调用
    * `bits`：类型是`class_data_bits_t`，用于存储类的数据（类的方法、属性、遵循的协议等信息），其长度在`64位`CPU下为8字节，是个指针，指向`class_rw_t *`

### 3.2 `class_rw_t`和`class_ro_t`总结

* `class_ro_t`存储了类在编译期确定的方法（包括其分类的）、成员变量、属性以及遵循的协议等信息，在运行时不会发生变化。编译期，类的`bits`指针指向的是`class_ro_t`指针（即此时的`class_rw_t *`实际上是`class_ro_t *`）。
    * 实例方法存储在类中
    * 类方法存储在元类中（【4.1】将给出证明）
* 在`realizeClassWithoutSwift()`执行之后，`class_rw_t`才会被初始化，同时存储类的方法、属性以及遵循的协议，实际上，`class_rw_t`和`class_ro_t`两者的方法列表（或属性列表、协议列表）的指针是相同的。
* 运行时向类动态添加属性、方法时，会修改`class_rw_t`的属性列表、方法列表指针，但`class_ro_t`对应的属性列表、方法列表不会变。

```!
一个待解决的坑：通过运行时添加方法（或属性、协议）改变了 class_rw_t 对应的方法列表（或属性列表、协议列表）的指针后，不知道为什么居然在 class_rw_t 的方法列表（或属性列表、协议列表）上找不到新增的方法（或属性、协议）了。这个问题困扰笔者好久了，在这里非常欢迎同学在评论区留言讨论。
```

## 4. 补充

### 4.1 类方法的存储位置

（Person类的类方法`printMyClassName()`）

````c
// 1. 获取 Person元类
(lldb) x/4gx pcls
0x100002820: 0x001d8001000027f9 0x0000000100b39140
0x100002830: 0x00000001003dc250 0x0000000000000000
(lldb) p/x 0x001d8001000027f9 & 0x00007ffffffffff8
(long) $50 = 0x00000001000027f8
(lldb) po 0x00000001000027f8
Person  // Person元类

// 2. 获取 Person元类 的 bits
(lldb) x/5gx 0x00000001000027f8
0x1000027f8: 0x001d800100b390f1 0x0000000100b390f0
0x100002808: 0x0000000102237440 0x0000000100000003
0x100002818: 0x00000001022373a0 // Person元类 的 bits

// 3. 获取 Person元类 的 class_rw_t
(lldb) p (class_rw_t *)(0x00000001022373a0 & 0x00007ffffffffff8)
(class_rw_t *) $52 = 0x00000001022373a0

// 4. 验证 Person元类 的 methods
(lldb) p $52->methods
(method_array_t) $55 = {
  list_array_tt<method_t, method_list_t> = {
     = {
      list = 0x0000000100002270
      arrayAndFlag = 4294976112
    }
  }
}
(lldb) p (method_list_t *)0x0000000100002270
(method_list_t *) $56 = 0x0000000100002270
(lldb) p *$56
(method_list_t) $57 = {
  entsize_list_tt<method_t, method_list_t, 3> = {
    entsizeAndFlags = 26
    count = 1
    first = {
      name = "printMyClassName" // 成功找到 Person类的类方法
      types = 0x0000000100001e96 "v16@0:8"
      imp = 0x00000001000014d0 (CCTest`+[Person printMyClassName] at main.m:37)
    }
  }
}
````
结论：**类方法 存储 在类的元类上，且位于元类的`class_ro_t`的`baseMethodList`指针上（或在`class_rw_t`的`methods`指针上）**

## 参考资料

[深入解析 ObjC 中方法的结构](https://github.com/draveness/analyze/blob/master/contents/objc/深入解析%20ObjC%20中方法的结构.md)（by Draveness）

## PS

* 源码工程已放到`github`上，请戳 [objc4-756.2源码](https://github.com/ConstantCody/objc4-756.2Demo)
