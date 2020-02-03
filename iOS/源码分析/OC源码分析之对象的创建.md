[此文在掘金上的地址](https://juejin.im/post/5de08bf85188254fc26bc242)

## 前言

> 想要成为一名`iOS开发高手`，免不了阅读源码。以下是笔者在`OC源码探索`中梳理的一个小系列——**类与对象篇**，欢迎大家阅读指正，同时也希望对大家有所帮助。
>
> 1. [OC源码分析之对象的创建](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之对象的创建.md)
> 2. [OC源码分析之isa](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之isa.md)
> 3. [OC源码分析之类的结构解读](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之类的结构解读.md)
> 4. 未完待续...

## 再前言：一个问题

进入主题之前，先请大家思考一下下面代码的输出

````c
#import <Foundation/Foundation.h>

@interface Person : NSObject

@end

@implementation Person

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [Person alloc];
        Person *p1 = [p init];
        Person *p2 = [p init];
        CCNSLog(@"p ==> %@", p);
        CCNSLog(@"p1 ==> %@", p1);
        CCNSLog(@"p2 ==> %@", p2);
    }
    return 0;
}
````
执行的结果是：

![](https://user-gold-cdn.xitu.io/2020/1/3/16f6830fef80bf4b?w=872&h=224&f=png&s=35598)

显而易见，对象p、p1、p2的内存地址一致，即这三者是同一个对象。那么问题来了，为什么这三个对象地址是一样的？`alloc`和`init`底层到底做了什么？带着这些问题，我们从源码的角度探索一下吧。

## 1. alloc源码分析

### 1.0 准备工作

1. 从 [苹果官方开源代码列表](https://opensource.apple.com/tarballs) 找到 `objc4`源码。

> 博主用到是最新版（[objc4-756.2源码](https://opensource.apple.com/tarballs/objc4/)），同时，`XCode`版本是`Version 11.3 (11C29)`。    
> 源码版本和`XCode`版本不需要与博主一致~

2. 下载到本地后，需要对工程进行一番编译调试，具体步骤可参考 **Cooci大佬** 的博客
[iOS_objc4-756.2 最新源码编译调试](https://juejin.im/post/5d9c829df265da5ba46f49c9)。

3. 编译通过后，就可以新建个target耍耍了。

![](https://user-gold-cdn.xitu.io/2020/1/4/16f6e7a7aa7829cc?w=2248&h=960&f=png&s=801017)

> 博主已经把编译好的`objc4-756.2`项目传到 [github](https://github.com/ConstantCody/objc4-756.2Demo) 了，感兴趣的同学可以下载哈~

因为`oc`语言的`runtime`特性，我们并不能肯定入口一定是`+alloc`方法，也就是说首先需要找到真正的入口。

常用的代码跟踪方式：
1. XCode菜单栏依次点击`Debug`->`Debug Workflow`->`Always show Disassembly`
2. `control` + `step into`
3. 下符号断点，如`alloc`

> 博主常用第一种，无他，手熟尔

### 1.1 `objc_alloc`——`alloc`的真正入口

给`[Person alloc]`加断点
![](https://user-gold-cdn.xitu.io/2020/1/3/16f695b987e20cc8?w=1318&h=402&f=png&s=113921)

此时，在XCode的菜单栏依次点击`Debug`->`Debug Workflow`->`Always show Disassembly`，得到汇编代码
![](https://user-gold-cdn.xitu.io/2020/1/3/16f6a2c9c57b8e0d?w=1756&h=912&f=png&s=366723)

不难发现，接下来会执行`objc_alloc`。源码如下图：

![](https://user-gold-cdn.xitu.io/2020/1/3/16f6a5219b0fe15b?w=1950&h=548&f=png&s=458655)

> 思考：为什么`[Person alloc]`会调用`objc_alloc`？（答案会在文末揭晓）

### 1.2 `callAlloc`分析——第一次的亲密接触

`objc_alloc()`内部调用`callAlloc()`，其源码为：

````c
// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
````

**对`callAlloc()`的分析如下：**
1. `slowpath(bool)`与`fastpath(bool)`：常用于`if-else`，可以优化判断的速度。

````c
// fastpath(x)：表示x为1（执行if代码块）的可能性更大
#define fastpath(x) (__builtin_expect(bool(x), 1))
// slowpath(x)：表示x为0（执行else代码块）的可能性更大
#define slowpath(x) (__builtin_expect(bool(x), 0))
````

2. `hasCustomAWZ()`：意思是`hasCustomAllocWithZone`，即是否有重写类的`+allocWithZone:`方法，但是它的值并不能简单地这么判断！先看源码

````c
bool hasCustomAWZ() {
    return ! bits.hasDefaultAWZ();
}
````

> **注意：`hasCustomAWZ()`的值问题**
>   * 类的`+initialize:`方法主要用于初始化静态变量。在其执行之前，`hasDefaultAWZ()`值为`false`，即`hasCustomAWZ()`为`true`；其执行之后，如果当前类重写了`+allocWithZone:`方法，`hasCustomAWZ()`为`true`，否则为`false`。
>   * 类的`+initialize:`方法会在第一次初始化该类之前调用。当调用`[cls alloc]`时，会触发`objc_msgSend`，然后会执行`+initialize:`。（感兴趣的同学可以分别打印`+alloc`和`+initialize:`方法加以验证）

**因此，当类第一次来到`callAlloc()`时，最终会执行`[cls alloc]`。**

3. `canAllocFast()`源码如下：

````c
bool canAllocFast() {
    assert(!isFuture());
    return bits.canAllocFast();
}
````

再往底层找`bits.canAllocFast()`，发现关键宏`FAST_ALLOC`

````c
#if FAST_ALLOC
    ...
    bool canAllocFast() {
        return bits & FAST_ALLOC;
    }
#else
    ...
    bool canAllocFast() {
        return false;
    }
#endif
````

继续深入，来到了`FAST_ALLOC`宏定义之处

````c
#if !__LP64__   // 当前操作系统不是64位
...
#elif 1         // 当前操作系统是64位
...
#else
...
#define FAST_ALLOC              (1UL<<2)
...
#endif
````

从上面宏代码可以得出这样的结论，即无论当前操作系统是不是64位，都没有定义`FAST_ALLOC`，也就是说，`canAllocFast()`永远是`false`!

因此，如果`hasCustomAWZ()`为`false`时，会直接去到`class_createInstance()`。

### 1.3 `alloc`->`_objc_rootAlloc`->`callAlloc`->`class_createInstance`

通过对`hasCustomAWZ()`的分析，我们知道类的第一次初始化最终是走到`callAlloc`的最后，即`return [cls alloc];`

1. 由于执行了`[cls alloc]`，这次真的来到`alloc()`方法了

````c
+ (id)alloc {
    return _objc_rootAlloc(self);
}
````

2. 接着是`_objc_rootAlloc()`

````c
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
````

3. 然后是`callAlloc()`->`class_createInstance()`

再次来到`callAlloc`，此时`hasCustomAWZ()`的值取决于当前类是否重写了`+allocWithZone:`方法。

由于`Person`类没有重写，`fastpath(!cls->ISA()->hasCustomAWZ())`为true，而`canAllocFast()`永远为`false`。

因此，接下来会走到`class_createInstance()`，其源码如下：

````c
id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}
````

### 1.4 `_class_createInstanceFromZone`

顾名思义，这是要创建对象！但是，`alloc`的时候就创建对象？？？现在，我们暂时把疑问放下，先分析一下源码：

````c
static __attribute__((always_inline)) 
id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    if (!cls) return nil;

    assert(cls->isRealized());

    // 一次读取类的信息位以提高性能
    bool hasCxxCtor = cls->hasCxxCtor();    // 是否有构造函数
    bool hasCxxDtor = cls->hasCxxDtor();    // 是否有析构函数
    bool fast = cls->canAllocNonpointer();
    
    // 计算内存
    size_t size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (!zone  &&  fast) {
        // 分配1块大小为size的连续内存
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        // 初始化对象的isa
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        if (zone) {
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else {
            obj = (id)calloc(1, size);
        }
        if (!obj) return nil;

        // Use raw pointer isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls);
    }

    return obj;
}
````

**对`_class_createInstanceFromZone()`的分析如下：**

1. `cls->instanceSize(extraBytes)`计算内存，此时的`extraBytes`是`0`，其源码是

````c
// 1.
size_t instanceSize(size_t extraBytes) {
    size_t size = alignedInstanceSize() + extraBytes;
    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;
    return size;
}

// 2.
uint32_t alignedInstanceSize() {
    return word_align(unalignedInstanceSize());
}

// 3. 字节对齐
static inline uint32_t word_align(uint32_t x) {
    return (x + WORD_MASK) & ~WORD_MASK;
}
static inline size_t word_align(size_t x) {
    return (x + WORD_MASK) & ~WORD_MASK;
}

// 4.
#ifdef __LP64__
#   define WORD_SHIFT 3UL
#   define WORD_MASK 7UL
#   define WORD_BITS 64
#else
#   define WORD_SHIFT 2UL
#   define WORD_MASK 3UL
#   define WORD_BITS 32
#endif

````

可见，`WORD_MASK`在`64`位系统下是`7`，否则是`3`，因此，`word_align()`在`64`位系统下是`8字节对齐`，否则是`4字节对齐`。

同时，`instanceSize()`函数又对内存大小又进行了最小`16字节`的限制。

2. `canAllocNonpointer()`是对isa的类型的区分，在 `__OBJC2__` 中，如果一个类使用`isa_t`类型的`isa`的话，`fast`就是`true`；而在`__OBJC2__`中，`zone`会被忽略，所以`!zone`也是`true`；

综上，接着就是`calloc()`和`initInstanceIsa()`。

3. `calloc()`的底层源码是在 [苹果开源的**libmalloc**](https://opensource.apple.com/tarballs/libmalloc/)
中，经过断点跟踪，发现`calloc`分配的内存大小受`segregated_size_to_fit()`影响，看下面源码：

````c
static MALLOC_INLINE size_t
segregated_size_to_fit(nanozone_t *nanozone, size_t size, size_t *pKey)
{
	size_t k, slot_bytes;

	if (0 == size) {
	    // Historical behavior
	    size = NANO_REGIME_QUANTA_SIZE;
	}
	// round up and shift for number of quanta
	k = (size + NANO_REGIME_QUANTA_SIZE - 1) >> SHIFT_NANO_QUANTUM; 
	// multiply by power of two quanta size
	slot_bytes = k << SHIFT_NANO_QUANTUM;							
	// Zero-based!
	*pKey = k - 1;													

	return slot_bytes;
}

#define SHIFT_NANO_QUANTUM	    4
#define NANO_REGIME_QUANTA_SIZE	    (1 << SHIFT_NANO_QUANTUM)	// 16
````

从代码可以看出，`slot_bytes`相当于`(size + 16-1) >> 4 << 4`，也就是`16字节对齐`，因此`calloc()`分配的内存大小必然是`16字节`的整数倍。

4. `initInstanceIsa()`就是初始化`isa`，并且关联`cls`。

```!
isa 是 objc 类结构中极其重要的一环，关于它的结构、初始化过程、继承关系等内容，博主会另起一篇文章讲述，敬请期待。
```

从上面的代码可以看出，`_class_createInstanceFromZone()`做了很多事情，并且最终确实创建了对象，几乎干了所有事情，那么，`init`又到底做了什么呢？请接着看下去。

## 2. init和new

### 1. `init`
````c
- (id)init {
    return _objc_rootInit(self);
}

id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
````
非常简单，`init`仅仅是将`alloc`创建的对象返回。为什么这样设计呢？其实并不难理解，在平时的开发中，我们常常会根据业务需求重写`init`，进行一些自定义的配置。

> `NSObject`的`init`是一种工厂设计方案，方便子类重写。

### 2. `new`

我们再看看`new`

````
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
````
很明显，`new`相当于`alloc`+`init`。

## 3. 总结

关于`alloc`、`init`以及`new`的源码分析就到这了。在`alloc`的过程中，`callAlloc`和`_class_createInstanceFromZone`这两个函数是重点。

> 以上源码流程分析，是建立在`objc4-756.2`源码的基础上的，`756.2`是目前最新的版本。

下面用流程图总结一下`alloc`创建对象的过程

![](https://user-gold-cdn.xitu.io/2020/1/5/16f761c270c257de?w=1744&h=1490&f=png&s=165001)

## 4. 结束语

以上就是OC对象源码创建的全部内容了。回首整个过程，有顺利也有坎坷，总体比较烧脑，但是经过`alloc`这一条龙服务后，仿佛完成了某项重任，身心无比愉悦。

OC源码分析之路，必将是荣誉之路，希望大家且行且珍惜，你我共勉！

### 补充

1. Q：思考：为什么`[Person alloc]`会调用`objc_alloc`？    
A：项目编译的时候，会读取镜像文件，在`_read_images()`函数中，有这样一段代码：

````
void _read_images(...)
{
...
#if SUPPORT_FIXUP
    // Fix up old objc_msgSend_fixup call sites
    for (EACH_HEADER) {
        message_ref_t *refs = _getObjc2MessageRefs(hi, &count);
        if (count == 0) continue;
        ...
        for (i = 0; i < count; i++) {
            fixupMessageRef(refs+i);
        }
    }
    ts.log("IMAGE TIMES: fix up objc_msgSend_fixup");
#endif
...
}
````

而在`fixupMessageRef()`中，有对`SEL_alloc`进行`IMP`的修复绑定

````
static void 
fixupMessageRef(message_ref_t *msg)
{    
    msg->sel = sel_registerName((const char *)msg->sel);

    if (msg->imp == &objc_msgSend_fixup) { 
        if (msg->sel == SEL_alloc) {
            msg->imp = (IMP)&objc_alloc;
        } else if (msg->sel == SEL_allocWithZone) {
            msg->imp = (IMP)&objc_allocWithZone;
        } else if (msg->sel == SEL_retain) {
            msg->imp = (IMP)&objc_retain;
        } else if (msg->sel == SEL_release) {
            msg->imp = (IMP)&objc_release;
        } else if (msg->sel == SEL_autorelease) {
            msg->imp = (IMP)&objc_autorelease;
        } else {
            msg->imp = &objc_msgSend_fixedup;
        }
    } 
...
}
````

通过`[Person alloc]`调用的是`objc_alloc()`这个既定事实，我们可以猜测，在项目编译生成`Mach-O`文件期间，形成了`SEL_alloc`与`objc_alloc`的对应关系。

> 大家可以将编译生成的`Mach-O`文件拖到`MachOView`中，验证一下，看看能否找到`objc_alloc`

![](https://user-gold-cdn.xitu.io/2020/1/5/16f7509ad74ac362?w=2048&h=1116&f=png&s=315588)

### 最后的问题

1. 下面的两次`alloc`，底层流程区别？如果`Person`类重写了`+allocWithZone:`呢？

````
Person *p1 = [Person alloc];
Person *p2 = [Person alloc];
````

> 大家可以自己试试，通过比较会帮助大家理解记忆`alloc`的流程。


