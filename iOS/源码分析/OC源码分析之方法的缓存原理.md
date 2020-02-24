[此文在掘金上的地址](https://juejin.im/post/5e49b929e51d4526d71d3946)

## 前言

> 想要成为一名`iOS开发高手`，免不了阅读源码。以下是笔者在`OC源码探索`中梳理的一个小系列——**类与对象篇**，欢迎大家阅读指正，同时也希望对大家有所帮助。
>
> 1. [OC源码分析之对象的创建](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之对象的创建.md)
> 2. [OC源码分析之isa](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之isa.md)
> 3. [OC源码分析之类的结构解读](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之类的结构解读.md)
> 4. [OC源码分析之方法的缓存原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的缓存原理.md)
> 5. 未完待续...

本文是针对 **方法缓存——`cache_t`** 的分析（且源码版本是 [objc4-756.2](https://opensource.apple.com/tarballs/objc4/)），下面进入正文。

## 1. `cache_t`源码分析

当你的`OC`项目编译完成后，类的实例方法（方法编号`SEL` 和 函数地址`IMP`）就保存在类的方法列表中。我们知道 `OC` 为了实现其动态性，将 **方法的调用包装成了 `SEL` 寻找 `IMP` 的过程**。试想一下，如果每次调用方法，都要去类的方法列表（甚至父类、根类的方法列表）中查询其函数地址，势必会对性能造成极大的损耗。为了解决这一问题，`OC` 采用了方法缓存的机制来提高调用效率，也就是`cache_t`，其作用就是缓存已调用的方法。当调用方法时，`objc_msgSend`会先去缓存中查找，如果找到就执行该方法；如果不在缓存中，则去类的方法列表（包括父类、根类的方法列表）查找，找到后会将方法的`SEL`和`IMP`缓存到`cache_t`中，以便下次调用时能够快速执行。

### 1.1 `cache_t`结构

首先看一下`cache_t`的结构
````c
struct cache_t {
    struct bucket_t *_buckets;  // 缓存数组，即哈希桶
    mask_t _mask;               // 缓存数组的容量临界值
    mask_t _occupied;           // 缓存数组中已缓存方法数量

    ... // 一些函数
};

#if __LP64__
typedef uint32_t mask_t;
#else
typedef uint16_t mask_t;
#endif

struct bucket_t {
private:
#if __arm64__
    uintptr_t _imp;
    SEL _sel;
#else
    SEL _sel;
    uintptr_t _imp;
#endif
    ... // 一些方法
};
````

从上面源码不难看出，在`64`位CPU架构下，`cache_t`长度是16字节。单从结构来看，方法是缓存在`bucket_t`（又称哈希桶）中，接下来用个例子验证一下`cache_t`是否缓存了已调用的方法。

### 1.2 方法缓存的验证

1. 创建一个简单的`Person`类，代码如下
````c
@interface Person : NSObject

- (void)methodFirst;
- (void)methodSecond;
- (void)methodThird;

@end

@implementation Person

- (void)methodFirst {
    NSLog(@"%s", __FUNCTION__);
}

- (void)methodSecond {
    NSLog(@"%s", __FUNCTION__);
}

- (void)methodThird {
    NSLog(@"%s", __FUNCTION__);
}

@end
````

2. **方法调用前的`cache_t`**

在方法调用前打个断点，看看`cache_t`的缓存情况

![](https://user-gold-cdn.xitu.io/2020/2/17/17050d86059e8d45?w=1772&h=1106&f=png&s=320546)

**说明：**
* 从`objc_class`结构很容易推导得出，`0x1000011d8`是`cache_t`首地址。（对类的结构感兴趣的同学请戳 [OC源码分析之类的结构解读](https://juejin.im/post/5e2c018a5188254dc42da951)）
* 由于还没有任何方法调用，所以`_mask`和`_occupied`都是0

3. **方法调用后的`cache_t`**

执行`alloc`和`init`这两个方法后，`cache_t`变化如下

![](https://user-gold-cdn.xitu.io/2020/2/17/17050e19dca9c7b2?w=1754&h=1146&f=png&s=283103)

从上图可知，调用`init`后，`_mask`的值是3，`_occupied`则是1。`_buckets`指针的值（数组首地址）发生了变化（从`0x1003db250`变成`0x101700090`），同时缓存了`init`方法的`SEL`和`IMP`。

```!
思考：
1. alloc 方法调用后，缓存在哪里？
2. 为什么 init 方法不在 _buckets 第一个位置？ 
```

继续执行`methodFirst`，再看`cache_t`

![](https://user-gold-cdn.xitu.io/2020/2/17/17050e7a42f1fea4?w=1786&h=1112&f=png&s=345563)

此时，`_mask`的值是3（没发生变化），`_occupied`则变成了2，`_buckets`指针地址没变，增加缓存了`methodFirst`方法的`SEL`和`IMP`。

接着是执行`methodSecond`，且看

![](https://user-gold-cdn.xitu.io/2020/2/17/17050f4b80003468?w=1824&h=1018&f=png&s=352986)

显然，`_occupied`变成了3，而`_buckets`指针地址不改变，同时新增`methodSecond`的方法缓存。

最后执行`methodThird`后，再看`cache_t`变化

![](https://user-gold-cdn.xitu.io/2020/2/17/17050fcf4e4c3dbd?w=1820&h=1070&f=png&s=319776)

这次的结果就完全不同了。`_mask`的值变成7，`_occupied`则重新变成了1，而`_buckets`不仅首地址变了，之前缓存的`init`、`methodFirst`和`methodSecond`方法也没了，仅存在的只有新增的`methodThird`方法。看来，`cache_t`并非是如我们所愿的那样——调用一个方法就缓存一个方法。

```!
思考：之前缓存的方法（init、methodFirst 和 methodSecond）哪去了？
```

### 1.3 `cache_t`小结

让我们梳理一下上面的例子。在依次执行`Person`的实例方法`init`、`methodFirst`、`methodSecond`、`methodThird`后，`cache_t`变化如下

|调用的方法|_buckets|_mask|_occupied|
|-|-|-|-|
|未调用方法|空|0|0|
|init|init|3|1|
|init、methodFirst|init、methodFirst|3|2|
|init、methodFirst、methodSecond|init、methodFirst、methodSecond|3|3|
|init、methodFirst、methodSecond、methodThird|methodThird|7|1|

可见，**`cache_t`的确能实时缓存已调用的方法**。

上面的验证过程也可以帮助我们理解`cache_t`三个成员变量的意义。直接从单词含义上解析，`bucket`可译为桶（即哈希桶），用于装方法；`occupied`可译为已占有，表示已缓存的方法数量；`mask`可译为面具、掩饰物，乍看无头绪，但是注意到`cache_t`中有获取容量的函数（`capacity`），其源码如下
````c
struct cache_t {
    ...
    mask_t mask();
    mask_t capacity();
    ...
}

mask_t cache_t::mask() 
{
    return _mask; 
}

mask_t cache_t::capacity() 
{
    return mask() ? mask()+1 : 0; 
}
````
由此可以得出，如果`_mask`是0，说明未调用实例方法，即桶的容量为0；当`_mask`不等于0的时候，意味着已经调用过实例方法，此时桶的容量为`_mask + 1`。故，`_mask`从侧面反映了桶的容量。

## 2. `cache_t`的方法缓存原理

接下来，笔者将从方法的调用过程开始分析`cache_t`的方法缓存原理。

### 2.1 `cache_fill`

`OC`方法的本质是 **消息发送（即`objc_msgSend`），底层是通过方法的 `SEL` 查找 `IMP`**。调用方法时，`objc_msgSend`会去`cache_t`中查询方法的函数实现（这部分是由汇编代码实现的，非常高效），在缓存中找的过程暂且不表；当缓存中没有的时候，则去类的方法列表中查找，直至找到后，再调用`cache_fill`，目的是为了将方法缓存到`cache_t`中，其源码如下

````c
void cache_fill(Class cls, SEL sel, IMP imp, id receiver)
{
#if !DEBUG_TASK_THREADS
    mutex_locker_t lock(cacheUpdateLock);
    cache_fill_nolock(cls, sel, imp, receiver);
#else
    _collecting_in_critical();
    return;
#endif
}
````

> `objc_msgSend`的具体流程笔者将另起一文分析，这里不作赘述。

### 2.2 `cache_fill_nolock`

`cache_fill`又会来到`cache_fill_nolock`，这个函数的作用是将方法的`SEL`和`IMP`写入`_buckets`，同时更新`_mask`和`_occupied`。

其源码以及详细分析如下：

````c
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{
    cacheUpdateLock.assertLocked();

    // 如果类未初始化
    if (!cls->isInitialized()) return;

    // 在获取cacheUpdateLock之前，确保其他线程没有将该方法写入缓存
    if (cache_getImp(cls, sel)) return;

    // 获取 cls 的 cache_t指针
    cache_t *cache = getCache(cls);

    // newOccupied为新的方法缓存数，等于 当前方法缓存数+1
    mask_t newOccupied = cache->occupied() + 1;
    // 获取当前cache_t的总容量，即 mask+1
    mask_t capacity = cache->capacity();
    if (cache->isConstantEmptyCache()) {
        // 当第一次调用类的实例方法时（如本文的【1.2】例中的`init`）
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
    }
    else if (newOccupied <= capacity / 4 * 3) {
        // 新的方法缓存数 不大于 总容量的3/4，按原样使用，无需扩容
    }
    else {
        // 新的方法缓存数 大于 总容量的3/4，需要扩容
        cache->expand();
    }

    // 根据sel获取bucket，此bucket的sel一般为0（说明这个位置还没缓存方法），
    // 也可能与实参sel相等（hash冲突，可能性很低）
    bucket_t *bucket = cache->find(sel, receiver);
    // 当且仅当bucket的sel为0时，执行_occupied++
    if (bucket->sel() == 0) cache->incrementOccupied();
    // 更新bucket的sel和imp
    bucket->set<Atomic>(sel, imp);
}

// INIT_CACHE_SIZE 即为4
enum {
    INIT_CACHE_SIZE_LOG2 = 2,
    INIT_CACHE_SIZE      = (1 << INIT_CACHE_SIZE_LOG2)
};
````

从上面的源码不难看出，`cache_fill_nolock`主要是`cache_t`缓存方法的调度中心，在这里会
1. 决定执行`_buckets`的哪一种缓存策略（初始化后缓存、直接缓存、扩容后缓存，三者取一）；
2. 然后通过方法的`sel`找到一个`bucket`，并更新这个`bucket`的`sel`和`imp`。（如果这个`bucket`的`sel`为0，说明是个空桶，正好可以缓存方法，于是执行`_occupied++`）。

```!
思考：为什么扩容临界点是 3/4？
```

### 2.3 `reallocate`

在下面这两种情况下会执行`reallocate`：
* 一是第一次初始化`_buckets`的时候
* 另一种则是`_buckets`扩容的时候

我们来看一下`reallocate`做了哪些事情
````c
void cache_t::reallocate(mask_t oldCapacity, mask_t newCapacity)
{
    // 当且仅当`_buckets`中有缓存方法时，feeOld为true
    bool freeOld = canBeFreed();

    // 获取当前buckets指针，即_buckets
    bucket_t *oldBuckets = buckets();
    // 开辟新的buckets指针
    bucket_t *newBuckets = allocateBuckets(newCapacity);

    // Cache's old contents are not propagated. 
    // This is thought to save cache memory at the cost of extra cache fills.
    // fixme re-measure this

    assert(newCapacity > 0);
    assert((uintptr_t)(mask_t)(newCapacity-1) == newCapacity-1);

    // 将新buckets、新mask（newCapacity-1）分别赋值跟当前的 _buckets 和 _mask
    setBucketsAndMask(newBuckets, newCapacity - 1);
    
    if (freeOld) {
        // 释放旧的buckets内存空间
        cache_collect_free(oldBuckets, oldCapacity);
        cache_collect(false);
    }
}
````

`reallocate`完美解释了在例【1.2】中的几个情况：
* `init`执行完后，`_buckets`指针地址变了，`_mask`变成了3；
* `methodThird`执行完后，`_buckets`不仅指针地址变了，同时之前缓存的`init`、`methodFirst`和`methodSecond`方法也都不在了

注意，`_occupied`的变化是在回到`cache_fill_nolock`后发生的。

```!
思考：扩容后，为什么不直接把之前缓存的方法加入新的buckets中？
```

### 2.4 `expand`

从`cache_fill_nolock`源码来看，当新的方法缓存数（_occupied+1）大于总容量（_mask+1）时，会对`_buckets`进行扩容，也就是执行`expand`函数，其源码如下

````c
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    // 获取当前总容量，即_mask+1
    uint32_t oldCapacity = capacity();
    // 新的容量 = 旧容量 * 2
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask
        newCapacity = oldCapacity;
    }
    
    reallocate(oldCapacity, newCapacity);
}
````
这个函数非常简单，仅仅是计算好新的容量后，就去调用`reallocate`函数。需要注意的是：
* 在不超过`uint32_t`大小（4字节）时，每次扩容为原来的2倍
* 如果超过了`uint32_t`，则重新申请跟原来一样大小的`buckets`

### 2.5 `find`

在执行完相应的`buckets`策略后，接下来就需要找到合适的位置（`bucket`），以存储
方法的`SEL`和`IMP`。`find`具体做的事情就是根据方法的`SEL`，返回一个符合要求的`bucket`，同样上源码

````c
bucket_t * cache_t::find(SEL s, id receiver)
{
    assert(s != 0);
    // 获取当前buckets，即_buckets
    bucket_t *b = buckets();
    // 获取当前mask，即_mask
    mask_t m = mask();
    // 由 sel & mask 得出起始索引值
    mask_t begin = cache_hash(s, m);
    mask_t i = begin;
    do {
        // sel为0：说明 i 这个位置尚未缓存方法；
        // sel等于s：命中缓存，说明 i 这个位置已缓存方法，可能是hash冲突
        if (b[i].sel() == 0  ||  b[i].sel() == s) {
            return &b[i];
        }
    } while ((i = cache_next(i, m)) != begin);

    // hack
    // 找不到多余的哈希桶（出错的处理，打印问题）。一般不会走到这里！
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)s, cls);
}

static inline mask_t cache_hash(SEL sel, mask_t mask) 
{
    return (mask_t)(uintptr_t)sel & mask;
}

#if __arm__  ||  __x86_64__  ||  __i386__
// objc_msgSend has few registers available.
// Cache scan increments and wraps at special end-marking bucket.
#define CACHE_END_MARKER 1
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return (i+1) & mask;
}

#elif __arm64__
// objc_msgSend has lots of registers available.
// Cache scan decrements. No end marker needed.
#define CACHE_END_MARKER 0
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}

#else
#error unknown architecture
#endif
````

从源码可以发现，`find`找`bucket`的方式用到了`hash`的思想：以`_buckets`作为哈希桶，以`cache_hash`作为哈希函数，进行哈希运算后得出索引值`index`（本质是`xx & mask`，所以`index`最大值就是`_mask`的值）。由于索引值是通过`哈希运算`得出的，其结果自然是无序的，这也是为什么上例中`init`方法不在`_buckets`第一个位置的原因。

## 3. 多线程对方法缓存的影响

既然哈希桶的数量是在运行时动态增加的，那么在多线程环境下调用方法时，对方法的缓存有没有什么影响呢？且看下面的分析。

### 3.1 多线程同时读取缓存

在整个`objc_msgSend`函数中，为了达到最佳的性能，对方法缓存的读取操作是没有添加任何锁的。而多个线程同时调用已缓存的方法，并不会引发`_buckets`和`_mask`的变化，**因此多个线程同时读取方法缓存的操作是不会有安全隐患的**。

### 3.2 多线程同时写缓存

从源码我们知道在桶数量扩容和写桶数据之前，系统使用了一个全局的互斥锁（`cacheUpdateLock.assertLocked()`）来保证写入的同步处理，并且在锁住的范围内部还做了一次查缓存的操作（`if (cache_getImp(cls, sel)) return;`），这样就 **保证了哪怕多个线程同时写同一个方法的缓存也只会产生写一次的效果，即多线程同时写缓存的操作也不会有安全隐患**。

### 3.3 多线程同时读写缓存

这个情况就比较复杂了，我们先看一下`objc_msgSend`读缓存的代码（以 [arm64架构汇编](https://opensource.apple.com/source/objc4/objc4-723/runtime/Messengers.subproj/objc-msg-arm64.s.auto.html) 为例）
````
.macro CacheLookup
	// x1 = SEL, x16 = isa
	ldp	x10, x11, [x16, #CACHE]	// x10 = buckets, x11 = occupied|mask
	and	w12, w1, w11		// x12 = _cmd & mask
	add	x12, x10, x12, LSL #4	// x12 = buckets + ((_cmd & mask)<<4)

	ldp	x9, x17, [x12]		// {x9, x17} = *bucket
1:	cmp	x9, x1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: x12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	x12, x10		// wrap if bucket == buckets
	b.eq	3f
	ldp	x9, x17, [x12, #-16]!	// {x9, x17} = *--bucket
	b	1b			// loop

3:	// wrap: x12 = first bucket, w11 = mask
	add	x12, x12, w11, UXTW #4	// x12 = buckets+(mask<<4)

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	x9, x17, [x12]		// {x9, x17} = *bucket
1:	cmp	x9, x1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: x12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	x12, x10		// wrap if bucket == buckets
	b.eq	3f
	ldp	x9, x17, [x12, #-16]!	// {x9, x17} = *--bucket
	b	1b			// loop

3:	// double wrap
	JumpMiss $0
	
.endmacro
````

其中，`ldp`指令的作用是将数据从内存读取出来存到寄存器，第一个`ldp`代码会 **把`cache_t`中的`_buckets` 和 `_occupied | _mask`整个结构体成员分别读取到`x10`和`x11`两个寄存器中**，并且`CacheLookup`的后续代码没有再次读取`cache_t`的成员数据，而是一直使用`x10`和`x11`中的值进行哈希查找。由于CPU能保证单条指令执行的原子性，所以 **只要保证`ldp	x10, x11, [x16, #CACHE]`这段代码读取到的`_buckets`与`_mask`是互相匹配的（即要么同时是扩容前的数据，要么同时是扩容后的数据），那么多个线程同时读写方法缓存也是没有安全隐患的**。

#### 3.3.1 编译内存屏障

这里有个疑问，即系统是如何确保`_buckets`与`_mask`的这种一致性的呢？让我们看一下这两个变量的写入源码

````c
void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask)
{
    // objc_msgSend uses mask and buckets with no locks.
    // It is safe for objc_msgSend to see new buckets but old mask.
    // (It will get a cache miss but not overrun the buckets' bounds).
    // It is unsafe for objc_msgSend to see old buckets and new mask.
    // Therefore we write new buckets, wait a lot, then write new mask.
    // objc_msgSend reads mask first, then buckets.

    // ensure other threads see buckets contents before buckets pointer
    mega_barrier();

    _buckets = newBuckets;
    
    // ensure other threads see new buckets before new mask
    mega_barrier();
    
    _mask = newMask;
    _occupied = 0;
}
````

这段`C++`代码先修改`_buckets`，然后再更新`_mask`的值，为了确保这个顺序不被编译器优化，这里使用了`mega_baerrier()`来实现 **编译内存屏障（Compiler Memory Barrier）**。
> 如果不设置 `编译内存屏障` 的话，编译器有可能会优化代码先赋值`_mask`，然后才是赋值`_buckets`，两者的赋值之间，如果另一个线程执行`ldp x10, x11, [x16, #0x10]`指令，得到的就是`旧_buckets`和`新_mask`，进而出现内存数组越界引发程序崩溃。
>
> 而加入了`编译内存屏障`后，就算得到的是`新_buckets`和`旧_mask`，也不会导致程序崩溃。

**可见，借助编译内存屏障的技巧在一定的程度上可以实现无锁读写技术。**

> 对`内存屏障`感兴趣的同学可戳 [理解 Memory barrier（内存屏障）](https://blog.csdn.net/world_hello_100/article/details/50131497)

#### 3.3.2 内存垃圾回收

我们知道，在多线程读写方法缓存时，写线程可能会扩容`_buckets`（开辟新的`_buckets`内存，同时销毁旧的`_buckets`），此时，如果其他线程读取到的`_buckets`是旧的内存，就有可能会发生读内存异常而系统崩溃。为了解决这个问题，`OC`使用了两个全局数组`objc_entryPoints`、`objc_exitPoints`，分别保存所有会访问到`cache`的函数的起始地址、结束地址
````c
extern "C" uintptr_t objc_entryPoints[];
extern "C" uintptr_t objc_exitPoints[];
````
下面列出这些函数（同样以 [arm64架构汇编](https://opensource.apple.com/source/objc4/objc4-723/runtime/Messengers.subproj/objc-msg-arm64.s.auto.html) 为例）
````
.private_extern _objc_entryPoints
_objc_entryPoints:
	.quad   _cache_getImp
	.quad   _objc_msgSend
	.quad   _objc_msgSendSuper
	.quad   _objc_msgSendSuper2
	.quad   _objc_msgLookup
	.quad   _objc_msgLookupSuper2
	.quad   0

.private_extern _objc_exitPoints
_objc_exitPoints:
	.quad   LExit_cache_getImp
	.quad   LExit_objc_msgSend
	.quad   LExit_objc_msgSendSuper
	.quad   LExit_objc_msgSendSuper2
	.quad   LExit_objc_msgLookup
	.quad   LExit_objc_msgLookupSuper2
	.quad   0
````

当线程扩容哈希桶时，会先把旧的桶内存保存在一个全局的垃圾回收数组变量`garbage_refs`中，然后再遍历当前进程（在`iOS`中，一个进程就是一个应用程序）中的所有线程，查看是否有线程正在执行`objc_entryPoints`列表中的函数（原理是`PC寄存器`中的值是否在`objc_entryPoints`和`objc_exitPoints`这个范围内），如果没有则说明没有任何线程访问`cache`，可以放心地对`garbage_refs`中的所有待销毁的哈希桶内存块执行真正的销毁操作；如果有则说明有线程访问`cache`，这次就不做处理，下次再检查并在适当的时候进行销毁。

以上，**`OC 2.0`的`runtime`巧妙的利用了`ldp汇编指令`、编译内存屏障技术、内存垃圾回收技术等多种手段来解决多线程读写的无锁处理方案，既保证了安全，又提升了系统的性能。**

> 在这里，特别感谢 **欧阳大哥**！他的 [深入解构objc_msgSend函数的实现](https://www.jianshu.com/p/df6629ec9a25) 这篇博文在多线程读写方法缓存方面对笔者的解惑！欧阳大哥的文章系统地梳理了`objc_msgSend`函数的流程，强烈推荐大家一读！




## 4. 问题讨论

来到这里，相信大家对`cache_t`缓存方法的原理已经有了一定的理解。现在请看下面的几个问题：

### 4.1 类方法的缓存位置

**Q**：`Person`类调用`alloc`方法后，缓存在哪里？

**A**：缓存在 `Person`元类 的 `cache_t` 中。证明如下图

![](https://user-gold-cdn.xitu.io/2020/2/17/170511b2721d737e?w=1816&h=1288&f=png&s=416602)

### 4.2 `_mask`的作用

**Q**：请说明`cache_t`中`_mask`的作用

**A**：`_mask`从侧面反映了`cache_t`中哈希桶的数量（`哈希桶的数量 = _mask + 1`），保证了查找哈希桶时不会出现越界的情况。

**题解**：从上面的源码分析，我们知道`cache_t`在任何一次缓存方法的时候，哈希桶的数量一定是 **`>=4`且能被 4整除的**，`_mask`则等于哈希桶的数量-1，也就是说，**缓存方法的时候，`_mask`的二进制位上全都是1**。当循环查询哈希桶的时候，索引值是由`xx & _mask`运算得出的，因此索引值是小于哈希桶的数量的（`index <= _mask`，故`index < capacity`），也就不会出现越界的情况。

### 4.3 关于扩容临界点`3/4`的讨论

**Q**：为什么扩容临界点是3/4？

**A**：一般设定临界点就不得不权衡 **空间利用率** 和 **时间利用率** 。在 `3/4` 这个临界点的时候，空间利用率比较高，同时又避免了相当多的哈希冲突，时间利用率也比较高。

**题解**：扩容临界点直接影响循环查找哈希桶的效率。设想两个极端情况：

当临界点是1的时候，也就是说当全部的哈希桶都缓存有方法时，才会扩容。这虽然让开辟出来的内存空间的利用率达到100%，但是会造成大量的哈希冲突，加剧了查找索引的时间成本，导致时间利用率低下，这与高速缓存的目的相悖；

当临界点是0.5的时候，意味着哈希桶的占用量达到总数一半的时候，就会扩容。这虽然极大避免了哈希冲突，时间利用率非常高，却浪费了一半的空间，使得空间利用率低下。这种以空间换取时间的做法同样不可取；

两相权衡下，**当扩容临界点是3/4的时候，空间利用率 和 时间利用率 都相对比较高**。

### 4.4 缓存循环查找的死循环情况

**Q**：缓存循环查找哈希桶是否会出现死循环的情况？

**A**：不会出现。

**题解**：当哈希桶的利用率达到3/4的时候，下次缓存的时候就会进行扩容，即空桶的数量最少也会有总数的1/4，因此循环查询索引的时候，一定会出现命中缓存或者空桶的情况，从而结束循环。

## 5. 总结

通过以上例子的验证、源码的分析以及问题的讨论，现在总结一下`cache_t`的几个结论：

1. `cache_t`能缓存调用过的方法。
2. `cache_t`的三个成员变量中，
    * `_buckets`的类型是`struct bucket_t *`，也就是指针数组，它表示一系列的哈希桶（已调用的方法的`SEL`和`IMP`就缓存在哈希桶中），一个桶可以缓存一个方法。
    * `_mask`的类型是`mask_t`（`mask_t`在`64`位架构下就是`uint32_t`，长度为4个字节），它的值等于哈希桶的总数-1（`capacity - 1`），侧面反映了哈希桶的总数。
    * `_occupied`的类型也是`mask_t`，它代表的是当前`_buckets`已缓存的方法数。
3. 当缓存的方法数到达临界点（桶总数的3/4）时，下次再缓存新的方法时，首先会丢弃旧的桶，同时开辟新的内存，也就是扩容（扩容后都是全新的桶，以后每个方法都要重新缓存的），然后再把新的方法缓存下来，此时`_occupied`为1。
4. 当多个线程同时调用一个方法时，可分以下几种情况：
    * 多线程读缓存：读缓存由汇编实现，无锁且高效，由于并没有改变`_buckets`和`_mask`，所以并无安全隐患。
    * 多线程写缓存：`OC`用了个全局的互斥锁（`cacheUpdateLock.assertLocked()`）来保证不会出现写两次缓存的情况。
    * 多线程读写缓存：`OC`使用了`ldp汇编指令`、编译内存屏障技术、内存垃圾回收技术等多种手段来解决多线程读写的无锁处理方案，既保证了安全，又提升了系统的性能。

## 6. 参考资料

* [深入解构objc_msgSend函数的实现](https://www.jianshu.com/p/df6629ec9a25)
* [理解 Memory barrier（内存屏障）](https://blog.csdn.net/world_hello_100/article/details/50131497)

## PS

* 源码工程已放到`github`上，请戳 [objc4-756.2源码](https://github.com/ConstantCody/objc4-756.2Demo)
