[此文在掘金上的地址](https://juejin.im/post/5e49d39de51d4526fc749278)

## 前言

> 想要成为一名`iOS开发高手`，免不了阅读源码。以下是笔者在`OC源码探索`中梳理的一个小系列——**类与对象篇**，欢迎大家阅读指正，同时也希望对大家有所帮助。
>
> 1. [OC源码分析之对象的创建](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之对象的创建.md)
> 2. [OC源码分析之isa](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之isa.md)
> 3. [OC源码分析之类的结构解读](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之类的结构解读.md)
> 4. [OC源码分析之方法的缓存原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的缓存原理.md)
> 5. [OC源码分析之方法的查找原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的查找原理.md)
> 6. [OC源码分析之方法的解析与转发原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的解析与转发原理.md)

在`Objective-C`中，当编译器遇到一个方法调用时，它会将方法的调用变成以下函数中的一个：

**`objc_msgSend`、`objc_msgSend_stret`、`objc_msgSendSuper`和`objc_msgSendSuper_stret`。**

发送给对象的父类的消息（使用`super`关键字时）是使用`objc_msgSendSuper`发送的，其他消息是使用`objc_msgSend`发送的。如果是以数据结构体作为返回值的方法，则是使用`objc_msgSendSuper_stret`或`objc_msgSend_stret`发送的。

上面四个函数都用于发送消息，做一些准备工作，继而进行方法的查找、解析和转发。本文的主题是方法的查找，笔者将从方法调用开始，一步一步详细解读`objc_msgSend`函数的实现以及方法的查找流程。

下面直接进入正题。

> 需要注意的是，笔者用的源码是 [objc4-756.2](https://opensource.apple.com/tarballs/objc4/)。

## 1 `objc_msgSend`解析

### 1.1 举个栗子

嗯，一个简单的例子，代码如下
````c
@interface Person : NSObject

- (void)personInstanceMethod1;

@end

@implementation Person

- (void)personInstanceMethod1 {
    NSLog(@"%s", __FUNCTION__);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *person = [Person alloc];
        [person personInstanceMethod1];
    }
    return 0;
}
````

用`clang`命令重新编译`main.m`文件
````c
clang -rewrite-objc main.m -o main.cpp
````

打开`main.cpp`文件
````c
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        Person *person = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc"));
        ((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("personInstanceMethod1"));
    }
    return 0;
}
````

去除强转后就容易分辨多了
````c
Person *person = objc_msgSend(objc_getClass("Person"), sel_registerName("alloc"));
objc_msgSend(person, sel_registerName("personInstanceMethod1"));
````
发现`+alloc`和`personInstanceMethod1`这两个方法的调用，实际上就是调用`objc_msgSend`函数。

### 1.2 方法调用的本质

可以在`arm64.s`文件中找到`objc_msgSend`函数的说明
````c
id objc_msgSend(id self, SEL _cmd, ...)
````

其中，第一个参数`self`是调用者本身，它也是接收者；第二个参数`_cmd`是方法编号；剩下的可变参数列表是方法自己的参数。

> 简单说明一下：
> * `id`指的是`OC`对象，每个对象在内存的结构都是不确定的，但其首地址指向的是对象的`isa`，通过`isa`，在运行时就能获取到`objc_class`
> * `objc_class`表示对象的`Class`，它的结构在编译后就确定了
> * `SEL`表示选择器，通常可理解为一个字符串。`OC`在运行时维护着一张`SEL`表，将字符串相同的方法名映射到唯一一个`SEL`上
>   * 任意类的相同方法名映射的`SEL`都相同（可以把`SEL`近似地等同于方法名）
>   * 可以通过`sel_registerName(char *name)`这个`C`函数得到`SEL`，`OC`也提供了一个语法糖`@selector`用来方便的调用该函数
> * `IMP`是一个函数指针。`OC`中的方法最终都会转换成纯`C`的函数，`IMP`表示的就是这些函数的地址。

从上面的例子可以得出一个结论：**方法调用的本质是通过`objc_msgSend`函数，向调用者发送名为`SEL`的消息，找到具体的函数地址`IMP`，进而执行该函数**。

也就是说，下面的两段代码实际上效果是相同的

![](https://user-gold-cdn.xitu.io/2020/3/21/170fc7e01dc18d74?w=1524&h=632&f=png&s=169566)

```!
要想使用 objc_msgSend 函数，需要改一处设置。如下图
```
![](https://user-gold-cdn.xitu.io/2020/3/21/170fc83ccfbc3e06?w=1724&h=536&f=png&s=108354)

## 2 方法的查找

方法的查找，也叫消息的查找，它的准备工作是从`objc_msgSend`开始，准备就绪后，才会展开查找。

### 2.1 `objc_msgSend`源码分析

以`arm64`架构为例，`objc_msgSend`的源码以及解析如下

````c
    // ENTRY 表示函数入口
    ENTRY _objc_msgSend
    UNWIND _objc_msgSend, NoFrame

    // p0存储的是objc_msgSend的第一个参数（即接收者）
    // 在此对接收者进行非空判断
    cmp	p0, #0			    // nil check and tagged pointer check
    // 是否支持 Tagged Pointer，64位CPU架构下为1
#if SUPPORT_TAGGED_POINTERS 
    // 64位，且 p0 <= 0（le即less or equal），跳转到 LNilOrTagged
    b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
#else
    // 32位，且 p0 == 0（eq即equal），跳转到 LReturnZero
    b.eq	LReturnZero
#endif
    // 读取接收者（实例对象、类对象、元类对象）的isa到p13
    ldr	p13, [x0]		        // p13 = isa
    // 根据isa得到class
    GetClassFromIsa_p16 p13		// p16 = class
LGetIsaDone:
    CacheLookup NORMAL	    // calls imp or objc_msgSend_uncached

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
    // p0 == 0，即接收者为nil，跳转到 LReturnZero
    b.eq	LReturnZero		    // nil check

    // tagged
    adrp	x10, _objc_debug_taggedpointer_classes@PAGE
    add	x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF
    ubfx	x11, x0, #60, #4
    ldr	x16, [x10, x11, LSL #3]
    adrp	x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGE
    add	x10, x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGEOFF
    cmp	x10, x16
    b.ne	LGetIsaDone

    // ext tagged
    adrp	x10, _objc_debug_taggedpointer_ext_classes@PAGE
    add	x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF
    ubfx	x11, x0, #52, #8
    ldr	x16, [x10, x11, LSL #3]
    b	LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:
    // x0 is already zero
    mov	x1, #0
    movi	d0, #0
    movi	d1, #0
    movi	d2, #0
    movi	d3, #0
    ret
    
    // END_ENTRY 表示函数结束
    END_ENTRY _objc_msgSend
````

可以看出，`objc_msgSend`主要是获取接收者的`isa`。

> 思考：`objc_msgSend`为什么要用汇编编写？

### 2.2 `GetClassFromIsa_p16`

以下两种情况下会执行`GetClassFromIsa_p16`：
* 当系统是64位架构，接收者不是`Tagged Pointer`对象，`isa`也不是`nonpointer`的；
* 当系统不是64位架构，且接收者的`isa`非空

以上两种情况下会来到`GetClassFromIsa_p16`，其源码如下

````c
.macro GetClassFromIsa_p16 /* src */

#if SUPPORT_INDEXED_ISA     // armv7k or arm64_32
	// Indexed isa
	mov	p16, $0			// optimistically set dst = src
	tbz	p16, #ISA_INDEX_IS_NPI_BIT, 1f	// done if not non-pointer isa
	// isa in p16 is indexed
	adrp	x10, _objc_indexed_classes@PAGE
	add	x10, x10, _objc_indexed_classes@PAGEOFF
	ubfx	p16, p16, #ISA_INDEX_SHIFT, #ISA_INDEX_BITS  // extract index
	ldr	p16, [x10, p16, UXTP #PTRSHIFT]	// load class from array
1:

#elif __LP64__
	// 64-bit packed isa
	and	p16, $0, #ISA_MASK

#else
	// 32-bit raw isa
	mov	p16, $0

#endif

.endmacro
````

由上文可知`p16 = class`，`$0`是接收者`isa`，64位架构下通过`isa & ISA_MASK`得到真正的`isa`，其值是类或元类，这取决于接收者是实例对象还是类对象。

也就是说，**`objc_msgSend`的主要作用是拿到接收者的`isa`信息，如果有，则执行`CacheLookup`**：

````c
LGetIsaDone:
    CacheLookup NORMAL	    // calls imp or objc_msgSend_uncached
````

### 2.3 `CacheLookup`

> 经过`objc_msgSend`一番操作，此时`p1 = SEL，p16 = isa`，

`CacheLookup`的作用就是在缓存中查找方法实现，它有三种模式：`NORMAL`、`GETIMP`和`LOOKUP`。

先来看看`CacheLookup`源码
````c
.macro CacheLookup
	// p1 = SEL, p16 = isa
	ldp	p10, p11, [x16, #CACHE]	// p10 = buckets, p11 = occupied|mask
#if !__LP64__
	and	w11, w11, 0xffff	// p11 = mask
#endif
	and	w12, w1, w11		// x12 = _cmd & mask
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// wrap: p12 = first bucket, w11 = mask
	add	p12, p12, w11, UXTW #(1+PTRSHIFT)
		                        // p12 = buckets + (mask << 1+PTRSHIFT)

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// double wrap
	JumpMiss $0
	
.endmacro
````

`CacheLookup`操作的是类的`cache`成员变量，它的结构是`cache_t`，主要用于缓存调用的方法（想了解`cache_t`的请戳 [OC源码分析之方法的缓存原理](https://juejin.im/post/5e49b929e51d4526d71d3946)，这里不作赘述）。

对`CacheLookup`的分析如下：

1. **关于`ldp	p10, p11, [x16, #CACHE]`**

找到`CACHE`的定义处
````c
#define CACHE            (2 * __SIZEOF_POINTER__)
#define CLASS            __SIZEOF_POINTER__
````
64位CPU架构下，指针的长度为8字节，所以`CACHE`为16字节。对于`cache_t`结构，其中`buckets`指针为8字节，存放在p10；4字节的`mask`和4字节的`occupied`共同存放在p11，p11的低32位（即w11）存放的是`mask`。

2. **找到目标`bucket`**
````c
#if __LP64__    // arm64
...
#define PTRSHIFT 3
...

#else           // arm64_32
...
#define PTRSHIFT 2
...
````

通过`sel & mask`哈希计算得出索引值，再取到对应的`bucket`，接着将`bucket`的`imp`和`sel`分别存入p17、p9。

> 思考：为什么索引值要左移`1 + PTRSHIFT`位？

3. **`CacheHit`、`CheckMiss`和`JumpMiss`**

当找到`bucket`后，接下来的流程如下：
* 如果`bucket`的`sel`不等于方法的`sel`，则执行`{imp, sel} = *--bucket`，也就是遍历`buckets`中的每个`bucket`，分别与方法的`sel`作对比
* 如果`bucket`的`sel`等于方法的`sel`，则执行`CacheHit`，即直接返回并执行`imp`；
* 如果找到buckets的第一个`bucket`，则执行`JumpMiss`
* 如果`bucket`的`sel`等于0，即该`bucket`是空桶，则执行`CheckMiss`

接下来看一下`CacheHit`、`CheckMiss`和`JumpMiss`这三个函数的源码
````c
// CacheHit: x17 = cached IMP, x12 = address of cached IMP, x1 = SEL
.macro CacheHit
.if $0 == NORMAL
	TailCallCachedImp x17, x12, x1	// authenticate and call imp
.elseif $0 == GETIMP
	mov	p0, p17
	cbz	p0, 9f			// don't ptrauth a nil imp
	AuthAndResignAsIMP x0, x12, x1	// authenticate imp and re-sign as IMP
9:	ret				// return IMP
.elseif $0 == LOOKUP
	// No nil check for ptrauth: the caller would crash anyway when they
	// jump to a nil IMP. We don't care if that jump also fails ptrauth.
	AuthAndResignAsIMP x17, x12, x1	// authenticate imp and re-sign as IMP
	ret				// return imp via x17
.else
.abort oops
.endif
.endmacro

// CheckMiss
.macro CheckMiss
	// miss if bucket->sel == 0
.if $0 == GETIMP
	cbz	p9, LGetImpMiss
.elseif $0 == NORMAL
	cbz	p9, __objc_msgSend_uncached
.elseif $0 == LOOKUP
	cbz	p9, __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro

// JumpMiss
.macro JumpMiss
.if $0 == GETIMP
	b	LGetImpMiss
.elseif $0 == NORMAL
	b	__objc_msgSend_uncached
.elseif $0 == LOOKUP
	b	__objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro
````
关于这三个函数，总结如下：
* `CacheHit`函数在`NORMAL`模式下，会把找到的`IMP`返回并调用；在`GETIMP`、`LOOKUP`这两种模式下仅仅是返回`IMP`，并没有调用。
* `JumpMiss`、`CheckMiss`这两个函数在三种模式下的行为基本一致：
    * 在`NORMAL`模式下，均调用`__objc_msgSend_uncached`；
    * 在`GETIMP`模式下，均调用`LGetImpMiss`，返回`nil`；
    * 在`LOOKUP`模式下，均调用`__objc_msgLookup_uncached`；

### 2.4 `__objc_msgSend_uncached` 和 `__objc_msgLookup_uncached`

同样看源码

````c
// __objc_msgSend_uncached
STATIC_ENTRY __objc_msgSend_uncached
UNWIND __objc_msgSend_uncached, FrameWithNoSaves

// THIS IS NOT A CALLABLE C FUNCTION
// Out-of-band p16 is the class to search
	
MethodTableLookup
TailCallFunctionPointer x17

END_ENTRY __objc_msgSend_uncached

// __objc_msgLookup_uncached

STATIC_ENTRY __objc_msgLookup_uncached
UNWIND __objc_msgLookup_uncached, FrameWithNoSaves

// THIS IS NOT A CALLABLE C FUNCTION
// Out-of-band p16 is the class to search
	
MethodTableLookup
ret

END_ENTRY __objc_msgLookup_uncached
````

发现这两个函数的主要工作就是调用`MethodTableLookup`

### 2.5 `MethodTableLookup`

源码如下

````c
.macro MethodTableLookup
	
	// push frame
	SignLR
	stp	fp, lr, [sp, #-16]!
	mov	fp, sp

	// save parameter registers: x0..x8, q0..q7
	sub	sp, sp, #(10*8 + 8*16)
	stp	q0, q1, [sp, #(0*16)]
	stp	q2, q3, [sp, #(2*16)]
	stp	q4, q5, [sp, #(4*16)]
	stp	q6, q7, [sp, #(6*16)]
	stp	x0, x1, [sp, #(8*16+0*8)]
	stp	x2, x3, [sp, #(8*16+2*8)]
	stp	x4, x5, [sp, #(8*16+4*8)]
	stp	x6, x7, [sp, #(8*16+6*8)]
	str	x8,     [sp, #(8*16+8*8)]

	// receiver and selector already in x0 and x1
	mov	x2, x16
	bl	__class_lookupMethodAndLoadCache3

	// IMP in x0
	mov	x17, x0
	
	// restore registers and return
	ldp	q0, q1, [sp, #(0*16)]
	ldp	q2, q3, [sp, #(2*16)]
	ldp	q4, q5, [sp, #(4*16)]
	ldp	q6, q7, [sp, #(6*16)]
	ldp	x0, x1, [sp, #(8*16+0*8)]
	ldp	x2, x3, [sp, #(8*16+2*8)]
	ldp	x4, x5, [sp, #(8*16+4*8)]
	ldp	x6, x7, [sp, #(8*16+6*8)]
	ldr	x8,     [sp, #(8*16+8*8)]

	mov	sp, fp
	ldp	fp, lr, [sp], #16
	AuthenticateLR

.endmacro
````

`MethodTableLookup`函数主要保存部分寄存器的参数，然后就是调用`_class_lookupMethodAndLoadCache3`函数。来到这里，**就意味着消息的缓存查找流程正式结束，接下来就要去方法列表中查找了**。在方法列表中的查找流程是C\C++实现的，效率低于缓存查找，因此这个流程也叫做消息的慢速查找流程。

### 2.6 `_class_lookupMethodAndLoadCache3`

`_class_lookupMethodAndLoadCache3`是个简单的`C\C++函数`，它只有一行代码，即调用`lookUpImpOrForward`函数

````c
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{        
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
````

接下来重点分析`lookUpImpOrForward`函数。

### 2.7 `lookUpImpOrForward`

既然是从`_class_lookupMethodAndLoadCache3`过来的，显然`initialize`和`resolver`这两个参数的值是`YES`（`resolver`这个标志决定是否进行后面的动态方法解析），`cache`则是`NO`。下面分析一下`lookUpImpOrForward`的源码

````c
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    // 如果是从缓存过来，cache为NO；消息解析和转发的时候，需要过一遍缓存，此时为YES；
    // 对缓存进行查找是没有加锁的，进而提高缓存查找的性能
    if (cache) {
        // cache_getImp 也是汇编编写的
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    // runtimeLock is held during isRealized and isInitialized checking
    // to prevent races against concurrent realization.

    // runtimeLock is held during method search to make
    // method-lookup + cache-fill atomic with respect to method addition.
    // Otherwise, a category could be added but ignored indefinitely because
    // the cache was re-filled with the old value after the cache flush on
    // behalf of the category.
    // 加锁，防止多线程操作，保证方法查找以及缓存填充（cache-fill）的原子性，
    // 以及确保加锁之后的代码不会有新方法添加导致缓存被冲洗（flush）.
    runtimeLock.lock();
    checkIsKnownClass(cls);
    // 如果类还没有realize，需要先进行realize，加载信息（属性、方法、协议等的attach）
    // 一般懒加载的类会走此方法
    if (!cls->isRealized()) {
        cls = realizeClassMaybeSwiftAndLeaveLocked(cls, runtimeLock);
        // runtimeLock may have been dropped but is now locked again
    }

    // 如果类未初始化，需要初始化
    if (initialize && !cls->isInitialized()) {
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
        // runtimeLock may have been dropped but is now locked again

        // If sel == initialize, class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }


 retry:    
    runtimeLock.assertLocked();

    // Try this class's cache.
    // 在当前类的缓存中查找
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.
    // 在当前类的方法列表中查找
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            // 如果找到，先缓存一下
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;
        }
    }

    // Try superclass caches and method lists.
    // 在父类的 缓存和方法列表中 查找
    {
        unsigned attempts = unreasonableClassCount();
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            // Halt if there is a cycle in the superclass chain.
            if (--attempts == 0) {
                _objc_fatal("Memory corruption in class list.");
            }
            
            // Superclass cache.
            imp = cache_getImp(curClass, sel);
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                    // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
            }
            
            // Superclass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
        }
    }

    // No implementation found. Try method resolver once.
    // 在【类...根类】的【缓存+方法列表】中都没找到IMP，进行方法解析
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlock();
        resolveMethod(cls, sel, inst);
        runtimeLock.lock();
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // Use forwarding.
    // 消息的转发
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlock();

    return imp;
}
````

简单梳理一下。对于某个`cls`来说，查找`IMP`总是会先从`cls`的缓存中开始（调用`cache_getImp`函数）；如果没找到，才会去`cls`的方法列表中查找，也就是调用`getMethodNoSuper_nolock`函数。如果依然没找到，会去`cls`的父类（父类的父类，一直到根类）重复【缓存+方法列表】这个查找流程，直到找到为止。如果到根类后依然未找到，则会进入方法解析甚至消息转发的流程。

需要注意的是，**如果在当前类的缓存中没找到，但是在其方法列表（或其“父类...根类”的缓存或方法列表）中找到了`IMP`，需要进行一次是否是消息转发的判断，如果不是消息转发，那么就对当前类的缓存进行填充操作，方便下次的调用；如果是消息转发，就退出循环。**

```!
lookUpImpOrForward 这个函数可以说是消息的调度中心，它不仅包含消息的查找，还囊括了消息的解析和转发。碍于篇幅，本文仅介绍其消息查找方面的内容，其余内容将另启一文详细说明。
```

接下来分析一下`cache_getImp`和`getMethodNoSuper_nolock`这两个函数。

### 2.8 `cache_getImp`

`cache_getImp`也是汇编编写的，其源码如下：
````c
STATIC_ENTRY _cache_getImp

	GetClassFromIsa_p16 p0
	CacheLookup GETIMP

LGetImpMiss:
	mov	p0, #0
	ret

	END_ENTRY _cache_getImp
````
主要是处理`isa`，进而在缓存中查找`IMP`，需要注意的是，这次的缓存查找模式是`GETIMP`（`GETIMP`模式下，如果`CacheLookup`查找失败会执行`LGetImpMiss`）。
> 关于`GetClassFromIsa_p16`和`CacheLookup`前面已做过解析。

### 2.9 `getMethodNoSuper_nolock`

调用`getMethodNoSuper_nolock`函数的目的是检索`cls`的方法列表，查找名为`sel`的方法，找到则返回这个方法（`method_t`结构，该结构含有`IMP`）。其源码为：

````c
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    runtimeLock.assertLocked();

    assert(cls->isRealized());
    // fixme nil cls? 
    // fixme nil sel?

    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists)
    {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }

    return nil;
}
````
`cls->data()->methods`是`method_array_t`结构，它可能是一维数组，也可能是二维数组，对其从`beginLists()`到`endLists()`进行迭代，确保了每次迭代时总能得到一维的`method_t`数组，接下来就是调用`search_method_list`函数，对这个数组进行检索。

### 2.10 `search_method_list`

````c
static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        // 一般来说，mlist是有序的，由此对其进行二分查找
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // Linear search of unsorted method list
        // mlist是无序的，只好遍历匹配
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }

#if DEBUG
    // sanity-check negative results
    if (mlist->isFixedUp()) {
        for (auto& meth : *mlist) {
            if (meth.name == sel) {
                _objc_fatal("linear search worked when binary search did not");
            }
        }
    }
#endif

    return nil;
}
````

`__builtin_expect()`表示对结果的期望：多数都是`true`，也就是说，这里通常会执行`findMethodInSortedMethodList`函数，对`mlist`进行二分查找；否则将遍历查找。

需要注意的是，当方法列表的结构发生改变的时候，就会触发对列表的排序（注意是方法所在的列表，而不是`rw`中所有的方法列表，即如果是二维数组，只需要重新排列当前方法所在的列表即可）。在以下函数的调用中，会触发对`mlist`的排序：
* `methodizeClass`
* `attachCategories`
* `addMethod`
* `addMethods`

### 2.11 `log_and_fill_cache`

`log_and_fill_cache`函数主要是将找到的`IMP`填充到缓存中，方便下次的调用。下面的三种情况会调用这个函数：
* 在类（或转发类）的缓存中没找到`IMP`，但是在其方法列表中找到了；
* 在类（该类不是转发类）的“父类...根类”的缓存中找到了`IMP`；
* 在类的“父类...根类”的方法列表中找到了`IMP`

````c
static void
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (objcMsgLogEnabled) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    cache_fill (cls, sel, imp, receiver);
}

void cache_fill(Class cls, SEL sel, IMP imp, id receiver)
{
#if !DEBUG_TASK_THREADS
    mutex_locker_t lock(cacheUpdateLock);
    cache_fill_nolock(cls, sel, imp, receiver);
#else
    _collecting_in_critical();
    return;
#endif
````

`log_and_fill_cache`函数比较简单，它调用了`cache_fill`函数，而`cache_fill`函数又调用了`cache_fill_nolock`函数，也就是说，填充缓存的关键函数是`cache_fill_nolock`。

> 关于`cache_fill_nolock`函数的解析，感兴趣的同学请戳 [OC源码分析之方法的缓存原理](https://juejin.im/post/5e49b929e51d4526d71d3946)，笔者有详细分析，这里就不作赘述。

## 3 总结

来到这里，已经把消息的查找介绍完毕，是时候总结一下了。

### 3.1 方法调用的本质

* 方法的调用会被编译器翻译成 `objc_msgSend`、`objc_msgSendSuper`、`objc_msgSend_stret`和`objc_msgSendSuper_stret` 这四个函数之一，这四个函数都是由汇编代码实现的。
    * 如果是以数据结构体作为返回值的方法，最终会转换成相应的`objc_msgSend_stret`或`objc_msgSendSuper_stret`函数
    * 使用`super`关键字调用方法时，是使用`objc_msgSendSuper`发送的
    * 其他的方法调用是使用`objc_msgSend`函数发送消息的。
* 方法的调用是通过`objc_msgSend`（或`objc_msgSendSuper`，或`objc_msgSend_stret`，或`objc_msgSendSuper_stret`）函数，向调用者发送名为`SEL`的消息，找到具体的函数地址`IMP`，进而执行该函数。

### 3.2 方法的查找

方法的查找流程如下：
1. 从`objc_msgSend`源码开始，会先去 类（实例方法）\元类（类方法） 的缓存中查找，如果找到`IMP`则返回并调用，否则会去 类\元类 的方法列表中查找
2. “步骤1”中的 **“缓存+方法列表”** 的查找方案，会遍历类的继承体系（类、类的父类、...、根类），分别进行查找，直至找到`IMP`为止。
    * 如果在当前类的缓存中没找到，但是在其方法列表（或其“父类...根类”的缓存或方法列表）中找到了IMP，需要进行一次是否是消息转发的判断，如果不是消息转发，那么就对当前类的缓存进行填充操作，方便下次调用时的查找；如果是消息转发，则不会缓存到当前类中
3. 如果遍历结束后依然未找到`IMP`，则会启动消息的解析或转发。

## 4 问题讨论

### 4.1 `objc_msgSend`为什么要用汇编编写？

A：原因大致有以下几点

* `C`语言是静态语言，无法实现参数个数、类型未知的情况下跳转到另一个任意的函数实现的功能；而汇编的寄存器可以做到这一点
* 汇编执行效率比C语言的高
* 使用汇编可以有效防止系统函数被hook，因此更为安全。

### 4.2 为什么索引值要左移`1 + PTRSHIFT`位？

A：这个笔者没有在`objc4-756.2`源码中找到答案，但是在`objc4-779.1`源码版本的`cache_t`结构中，存在关于这个问题的解释。其部分源码如下：
````c
// How much the mask is shifted by.
static constexpr uintptr_t maskShift = 48;
    
// Additional bits after the mask which must be zero. msgSend
// takes advantage of these additional bits to construct the value
// `mask << 4` from `_maskAndBuckets` in a single instruction.
static constexpr uintptr_t maskZeroBits = 4;
    
// The largest mask value we can store.
static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;
    
// The mask applied to `_maskAndBuckets` to retrieve the buckets pointer.
static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;
````

总的来说，在64位中，`mask`的偏移值为48，也就是最高的16位存储`mask`；接着的44位是`buckets`指针地址，最低的4位是附加位，即 **`buckets`的有效指针地址仅仅是64位中的[4, 47]位**。在`CacheLookup`源码中，由`_cmd & mask`哈希运算可以得到索引值（`索引值 < ((1 << 16) - 1)`），如果想得到这个位置的`bucket`，其索引值必须左移4位后，才能与`buckets`指针地址相加得到正确的`bucket`地址。

## 5 参考资料

* [Objective-C 中的消息与消息转发](https://blog.ibireme.com/2013/11/26/objective-c-messaging)
* [Objective-C底层汇总](https://devyang.space/2019/12/30/Objective-C底层汇总/)

## 6 PS

* 源码工程已放到`github`上，请戳 [objc4-756.2源码](https://github.com/ConstantCody/objc4-756.2Demo)
* 你也可以自行下载 [苹果官方objc4源码](https://opensource.apple.com/tarballs/objc4/) 研究学习。
* 转载请注明出处！谢谢！
