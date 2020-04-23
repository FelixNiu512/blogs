[此文在掘金上的地址](https://juejin.im/post/5e8e95266fb9a03c7a331a68)

## 前言

> 想要成为一名`iOS开发高手`，免不了阅读源码。以下是笔者在`OC源码探索`中梳理的一个小系列——**类与对象篇**，欢迎大家阅读指正，同时也希望对大家有所帮助。
>
> 1. [OC源码分析之对象的创建](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之对象的创建.md)
> 2. [OC源码分析之isa](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之isa.md)
> 3. [OC源码分析之类的结构解读](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之类的结构解读.md)
> 4. [OC源码分析之方法的缓存原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的缓存原理.md)
> 5. [OC源码分析之方法的查找原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的查找原理.md)
> 6. [OC源码分析之方法的解析与转发原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的解析与转发原理.md)

`OC`中方法的调用是通过`objc_msgSend`（或`objc_msgSendSuper`，或`objc_msgSend_stret`，或`objc_msgSendSuper_stret`）函数，向调用者发送名为`SEL`的消息，找到具体的函数地址`IMP`，进而执行该函数。如果找不到`IMP`，会进行方法的解析，这相当于提供一次容错处理；方法解析之后，如果依然找不到`IMP`，还有最后一次机会，那就是消息的转发。

方法的查找流程尽在 [OC源码分析之方法的查找原理](https://github.com/ConstantCody/blogs/blob/master/iOS/源码分析/OC源码分析之方法的查找原理.md) 一文中，文接此文，本文将深入剖析方法的解析与转发。

下面进入正题。

> 需要注意的是，笔者用的源码是 [objc4-756.2](https://opensource.apple.com/tarballs/objc4/)。

## 1 方法的解析

方法的解析，即`method resolver`（又名消息的解析，也叫方法决议），其建立在方法的查找的失败结果上，入口源码如下：

````c
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
````

它主要是调用了`resolveMethod`函数。`resolveMethod`函数处理完毕之后，还要重新执行一次`retry`（再走一遍方法的查找流程）。其中，`triedResolver`这个变量使得消息的解析只进行一次。

### 1.1 `resolveMethod`

且看`resolveMethod`函数源码：

````c
static void resolveMethod(Class cls, SEL sel, id inst)
{
    runtimeLock.assertUnlocked();
    assert(cls->isRealized());

    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            resolveInstanceMethod(cls, sel, inst);
        }
    }
}
````

这里有两个分支，主要是对`cls`做个是否元类的判断：
* 不是元类，意味着调用的是实例方法，那么执行`resolveInstanceMethod`函数
* 是元类，说明调用的是类方法，执行`resolveClassMethod`函数，之后如果依然没找到`IMP`，则再去执行`resolveInstanceMethod`函数；

先看实例方法的情况

### 1.2 实例方法解析

`resolveInstanceMethod`源码如下：
````c
static void resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    runtimeLock.assertUnlocked();
    assert(cls->isRealized());

    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // 如果你没有实现类方法 +(BOOL)resolveInstanceMethod:(SEL)sel
        // NSObject也有实现，所以一般不会走这里
        // 注意这里传入的第一个参数是：cls->ISA()，也就是元类
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    // 调用类方法： +(BOOL)resolveInstanceMethod:(SEL)sel
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    // 再找一次imp（这次是sel，而不是resolveInstanceMethod）
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
````

`resolveInstanceMethod`函数先后调用了两次`lookUpImpOrNil`：
* 第一次的调用是判断类（包括其父类，直至根类）是否实现了`+(BOOL)resolveInstanceMethod:(SEL)sel`类方法
    * `SEL_resolveInstanceMethod`相当于`@selector(resolveInstanceMethod:)`，`NSObject`类中有实现这个类方法（返回的是`NO`，会影响是否打印），所以一般会接着往下走。
* 第二次的调用的目的是检测是否有`sel`对应的`IMP`。假如你在`+(BOOL)resolveInstanceMethod:(SEL)sel`中添加了`sel`的函数地址`IMP`，此时再次去查找这个`IMP`就能找到。

注意到这两次调用中，`resolver`都是`NO`，因此在其调用`lookUpImpOrForward`时不会触发 **消息的解析**，仅仅是从“类、父类、...、根类”的缓存中和方法列表中找`IMP`，没找到会触发 **消息转发**。

`lookUpImpOrNil`函数源码：
````c
IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
````

这里会判断`IMP`是否是消息转发而来的，如果是，就不返回。

### 1.3 类方法解析

类方法的解析首先是调用`resolveClassMethod`函数，其源码如下：

````c
// 这里的cls是元类，因为类方法存储在元类
static void resolveClassMethod(Class cls, SEL sel, id inst)
{
    runtimeLock.assertUnlocked();
    assert(cls->isRealized());
    assert(cls->isMetaClass());

    if (! lookUpImpOrNil(cls, SEL_resolveClassMethod, inst, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // 如果你没有实现类方法 +(BOOL)resolveClassMethod:(SEL)sel
        // NSObject也有实现，所以一般不会走这里
        // 注意这里的第一个参数是cls，是元类
        return;
    }

    Class nonmeta;
    {
        mutex_locker_t lock(runtimeLock);
        // 获取 元类的对象，即类。换句话说，nonmeta 也就是 inst
        nonmeta = getMaybeUnrealizedNonMetaClass(cls, inst);
        // +initialize path should have realized nonmeta already
        if (!nonmeta->isRealized()) {
            _objc_fatal("nonmeta class %s (%p) unexpectedly not realized",
                        nonmeta->nameForLogging(), nonmeta);
        }
    }
    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    // 调用类方法： +(BOOL)resolveClassMethod:(SEL)sel
    bool resolved = msg(nonmeta, SEL_resolveClassMethod, sel);

    // 再找一次imp
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveClassMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
````

你会发现，这个函数与`resolveInstanceMethod`函数大体相同，需要留意的是，这次判断类（包括其父类，直至根类）是否实现的是`+(BOOL)resolveClassMethod:(SEL)sel`类方法。

让我们回顾一下`resolveMethod`函数对类方法的解析

````c
// try [nonMetaClass resolveClassMethod:sel]
// and [cls resolveInstanceMethod:sel]
resolveClassMethod(cls, sel, inst);
if (!lookUpImpOrNil(cls, sel, inst, 
        NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
{
    // 此时的cls为元类，也就是 NSObject 调用 resolveInstanceMethod:
    resolveInstanceMethod(cls, sel, inst);
}
````

在经过`resolveClassMethod`的处理之后，如果依然没有找到类方法的`IMP`，就会再次执行`resolveInstanceMethod`函数！不同于实例方法的是，此时的`cls`是元类，因此`msg(cls, SEL_resolveInstanceMethod, sel);`即是向元类内部发送`resolveInstanceMethod:`消息，也就意味着是根类调用`resolveInstanceMethod:`方法（这次只能在根类的分类中补救了），同时缓存查找类方法的`IMP`仅发生在根元类和根类中，而方法列表中查找类方法的`IMP`则分别在“元类、元类的父类、...、根元类、根类”中进行。

简而言之，**当我们调用一个类方法时，如果在类中没有实现，同时在`resolveClassMethod`中也没有处理，那么最终会调用根类（`NSObject`）的同名实例方法**。

### 1.4 举个栗子

通过上述的分析，相信大家对方法的解析有了一定的认知，下面我们来整个简单的例子消化一下。

````c
@interface Person : NSObject

+ (void)personClassMethod1;
- (void)personInstanceMethod1;

@end

@implementation Person

@end
````

一个简单的`Person`类，里面分别有一个类方法和一个实例方法，但是都没有实现。

接着添加对这两个方法的解析：

````c
- (void)unimplementedMethod:(SEL)sel {
    NSLog(@"没实现？没关系，绝不崩溃");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSLog(@"动态实例方法解析：%@", NSStringFromSelector(sel));
    if (sel == @selector(personInstanceMethod1)) {
        IMP methodIMP = class_getMethodImplementation(self, @selector(unimplementedMethod:));
        Method method = class_getInstanceMethod(Person.class, @selector(unimplementedMethod:));
        const char *methodType = method_getTypeEncoding(method);
        return class_addMethod(Person.class, sel, methodIMP, methodType);
    }
    return [super resolveInstanceMethod:sel];
}

+ (BOOL)resolveClassMethod:(SEL)sel {
    NSLog(@"动态类方法解析：%@", NSStringFromSelector(sel));
    if (sel == @selector(personClassMethod1)) {
        IMP methodIMP = class_getMethodImplementation(self, @selector(unimplementedMethod:));
        Method method = class_getInstanceMethod(Person.class, @selector(unimplementedMethod:));
        const char *methodType = method_getTypeEncoding(method);
        return class_addMethod(objc_getMetaClass("Person"), sel, methodIMP, methodType);
    }
    return [super resolveClassMethod:sel];
}
````

看看打印：

![](https://user-gold-cdn.xitu.io/2020/4/13/17172b681b7198e5?w=1650&h=726&f=png&s=180356)

通过对类方法解析的源码分析，我们知道，也可以把对`Person`类方法的处理放在`NSObject`分类的`resolveClassMethod:`或`resolveInstanceMethod:`中，都能达到相同的效果（记得把`Person`类中的`resolveClassMethod:`处理去掉）。这里略过不提。

## 2 消息转发

方法的调用经过了查找、解析，如果还是没有找到`IMP`，就会来到消息转发流程。它的入口在`lookUpImpOrForward`函数靠后的位置
````c
// No implementation found, and method resolver didn't help.     
// Use forwarding.
imp = (IMP)_objc_msgForward_impcache;
cache_fill(cls, sel, imp, inst);
````

### 2.1 消息的转发起始和结束

`_objc_msgForward_impcache`是汇编函数，以`arm64`架构为例，其源码如下：

````c
STATIC_ENTRY __objc_msgForward_impcache

// No stret specialization.
b	__objc_msgForward

END_ENTRY __objc_msgForward_impcache
````

`__objc_msgForward_impcache`内部调用了`__objc_msgForward`
````c
ENTRY __objc_msgForward

adrp	x17, __objc_forward_handler@PAGE
ldr	p17, [x17, __objc_forward_handler@PAGEOFF]
TailCallFunctionPointer x17
	
END_ENTRY __objc_msgForward
````

这个函数主要做的事情是，通过页地址与页地址偏移的方式，拿到`_objc_forward_handler`函数的地址并调用。

> 说明：
> * `adrp`是以页为单位的大范围的地址读取指令，这里的`p`就是`page`的意思
> * `ldr`类似与`mov`和`mvn`，当立即数（`__objc_msgForward`中是`[x17, __objc_forward_handler@PAGEOFF]`，`PAGEOFF`是页地址偏移值）大于`mov`和`mvn`能操作的最大数时，就使用`ldr`。

在`OBJC2`中，`_objc_forward_handler`实际上就是`objc_defaultForwardHandler`函数，其源码如下：

````c
// Default forward handler halts the process.
__attribute__((noreturn)) void 
objc_defaultForwardHandler(id self, SEL sel)
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}
void *_objc_forward_handler = (void*)objc_defaultForwardHandler;
````

是不是很熟悉？当我们调用一个没实现的方法时，报的错就是“unrecognized selector sent to ...”

但是问题来了，说好的消息转发流程呢？这才刚开始怎么就结束了？不急，憋慌，且看下去。

### 2.2 消息转发的调用栈

回顾方法解析时举的例子，不妨把解析的内容去掉，**Let it crash!** 

![](https://user-gold-cdn.xitu.io/2020/4/13/17173478a9ff17a4?w=1888&h=1086&f=png&s=361375)

发现在崩溃之前与消息转发相关的内容是，调用了`_CF_forwarding_prep_0`和`___forwarding___`这两个函数。遗憾的是这两个函数并未开源。

既然崩溃信息不能提供帮助，只好打印具体的调用信息了。

在方法的查找流程中，`log_and_fill_cache`函数就跟打印有关，跟踪其源码：
````c
static void
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (slowpath(objcMsgLogEnabled && implementer)) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    cache_fill(cls, sel, imp, receiver);
}

bool objcMsgLogEnabled = false;

// Define SUPPORT_MESSAGE_LOGGING to enable NSObjCMessageLoggingEnabled
#if !TARGET_OS_OSX
#   define SUPPORT_MESSAGE_LOGGING 0
#else
#   define SUPPORT_MESSAGE_LOGGING 1
#endif
````

打印的关键函数就是`logMessageSend`，但是它受`SUPPORT_MESSAGE_LOGGING`和`objcMsgLogEnabled`控制。

继续跟进`SUPPORT_MESSAGE_LOGGING`
````c
#if !DYNAMIC_TARGETS_ENABLED
    #define TARGET_OS_OSX               1
    ...
#endif
    
#ifndef DYNAMIC_TARGETS_ENABLED
 #define DYNAMIC_TARGETS_ENABLED   0
#endif
````
从源码不难看出`TARGET_OS_OSX`的值是1，因此，`SUPPORT_MESSAGE_LOGGING`也为1！

如果能把`objcMsgLogEnabled`改成`true`，显然就可以打印调用信息了。通过全局搜索`objcMsgLogEnabled`，我们找到了`instrumentObjcMessageSends`这个关键函数
````c
void instrumentObjcMessageSends(BOOL flag)
{
    bool enable = flag;

    // Shortcut NOP
    if (objcMsgLogEnabled == enable)
        return;

    // If enabling, flush all method caches so we get some traces
    if (enable)
        _objc_flush_caches(Nil);

    // Sync our log file
    if (objcMsgLogFD != -1)
        fsync (objcMsgLogFD);

    objcMsgLogEnabled = enable;
}
````

接下来就好办了！来到`main.m`，添加以下代码

````c
extern void instrumentObjcMessageSends(BOOL flag);
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        instrumentObjcMessageSends(true);
        [Person personClassMethod1];
        instrumentObjcMessageSends(false);
    }
    return 0;
}
````

运行工程，直到再次崩溃。此时已打印函数调用栈，日志文件位置在`logMessageSend`函数中有标注
````c
    // Create/open the log file
    if (objcMsgLogFD == (-1))
    {
        snprintf (buf, sizeof(buf), "/tmp/msgSends-%d", (int) getpid ());
        objcMsgLogFD = secure_open (buf, O_WRONLY | O_CREAT, geteuid());
        if (objcMsgLogFD < 0) {
            // no log file - disable logging
            objcMsgLogEnabled = false;
            objcMsgLogFD = -1;
            return true;
        }
    }
````

打开`Finder`（访达），`cmd` + `shift` + `G`快捷键，输入`/tmp/msgSends`，找到最新的一份日志文件（数字最大）

![](https://user-gold-cdn.xitu.io/2020/4/14/171765ec112246e8?w=1530&h=566&f=png&s=145604)

打印结果如下：

![](https://user-gold-cdn.xitu.io/2020/4/14/1717662dd8a15430?w=1376&h=718&f=png&s=187188)

从这份日志可以看出，与转发相关的方法是`forwardingTargetForSelector`和`methodSignatureForSelector`，分别对应了消息的快速转发流程和慢速转发流程，接下来开始分析这两个方法。

### 2.3 消息的快速转发

`forwardingTargetForSelector:`对应的就是消息的快速转发流程，它在源码中只是简单的返回`nil`（可在子类或分类中重写）
````c
+ (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}

- (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}
````

不过我们可以在开发文档中找到说明（`cmd` + `shift` + `0`快捷键）

![](https://user-gold-cdn.xitu.io/2020/4/14/17176904fae24d23?w=1584&h=1440&f=png&s=369872)

概括地说，`forwardingTargetForSelector:`主要是返回一个新的`receiver`，去处理`sel`这个当前类无法处理的消息，如果处理不了，会转到效率低下的`forwardInvocation:`。在效率方面，`forwardingTargetForSelector:`领先`forwardInvocation:`一个数量级，因此，最好不要用后者的方式处理消息的转发逻辑。

关于`forwardingTargetForSelector:`返回的新的`receiver`，需要注意一下几点：
* 绝对不能返回`self`，否则会陷入无限循环；
* 不处理的话，可以返回`nil`，或者`[super forwardingTargetForSelector:sel]`（非根类的情况），此时会走`methodSignatureForSelector:`慢速转发流程；
* 如果有这个`receiver`，此时相当于执行`objc_msgSend(newReceiver, sel, ...)`，那么它必须拥有和被调用的方法相同方法签名的方法（方法名、参数列表、返回值类型都必须一致）。

#### 2.3.1 举个栗子

我们可以实验一下，准备工作如下
````c
@interface ForwardObject : NSObject

@end

@implementation ForwardObject

+ (void)personClassMethod1 {
    NSLog(@"类方法转发给%@，执行%s", [self className], __FUNCTION__);
}

- (void)personInstanceMethod1 {
    NSLog(@"实例方法转发给%@，执行%s", [self className], __FUNCTION__);
}

@end

@interface Person : NSObject

+ (void)personClassMethod1;
- (void)personInstanceMethod1;

@end

@implementation Person

- (id)forwardingTargetForSelector:(SEL)aSelector {
    NSLog(@"实例方法开始转发");
    return [ForwardObject alloc];
}

+ (id)forwardingTargetForSelector:(SEL)sel {
    NSLog(@"类方法开始转发");
    return [ForwardObject class];
}

@end
````

显然，`ForwardObject`作为消息转发后的处理类，拥有`Person`类的同名类方法和实例方法。现在开始验证，结果如下：

![](https://user-gold-cdn.xitu.io/2020/4/14/17176e5e998dce02?w=1904&h=994&f=png&s=388661)

事实证明确实有效！接下来看消息的慢速转发流程。

### 2.4 消息的慢速转发

如果`forwardingTargetForSelector:`没有处理消息（如返回`nil`），就会启动`慢速转发流程`，也就是`methodSignatureForSelector:`方法，同样需要在子类或分类中重写
````c
// Replaced by CF (returns an NSMethodSignature)
+ (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    _objc_fatal("+[NSObject methodSignatureForSelector:] "
                "not available without CoreFoundation");
}

// Replaced by CF (returns an NSMethodSignature)
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    _objc_fatal("-[NSObject methodSignatureForSelector:] "
                "not available without CoreFoundation");
}
````

通过阅读官方文档，我们得出以下结论：
* `methodSignatureForSelector:`方法是跟`forwardInvocation:`方法搭配使用的，前者需要我们根据`sel`返回一个方法签名，后者会把这个方法签名封装成一个`NSInvocation`对象，并将其作为形参。
* 如果有目标对象能处理`Invocation`中的`sel`，`Invocation`可以指派这个对象处理；否则不处理。
    * `Invocation`可以指派多个对象处理

> 注意：消息的慢速转发流程性能较低，如果可以的话，你应该尽可能早地处理掉消息（如在方法解析时，或在消息的快速转发流程时）。

#### 2.4.1 举个栗子

针对慢速流程，同样可以验证。这里把快速转发例子中的`Person`类修改一下：
````c
@implementation Person

// MARK: 慢速转发--类方法

+ (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSLog(@"类方法慢速转发：%s, sel：%@", __FUNCTION__, NSStringFromSelector(aSelector));
    if (aSelector == @selector(personClassMethod1)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

+ (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL aSelector = [anInvocation selector];
    NSLog(@"类方法慢速转发：%s, sel：%@", __FUNCTION__, NSStringFromSelector(aSelector));
    id target = [ForwardObject class];
    if ([target respondsToSelector:aSelector]) [anInvocation invokeWithTarget:target];
    else [super forwardInvocation:anInvocation];
}

// MARK: 慢速转发--实例方法

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSLog(@"实例方法慢速转发：%s, sel：%@", __FUNCTION__, NSStringFromSelector(aSelector));
    if (aSelector == @selector(personInstanceMethod1)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL aSelector = [anInvocation selector];
    NSLog(@"实例方法慢速转发：%s, sel：%@", __FUNCTION__, NSStringFromSelector(aSelector));
    ForwardObject *obj = [ForwardObject alloc];
    if ([obj respondsToSelector:aSelector]) [anInvocation invokeWithTarget:obj];
    else [super forwardInvocation:anInvocation];
}

@end
````
其结果如下图所示，显然也没有崩溃。

![](https://user-gold-cdn.xitu.io/2020/4/14/1717764ba33891bd?w=2002&h=1130&f=png&s=486536)

> 对方法签名类型编码不熟悉的可以查看 [苹果官方的类型编码介绍](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

## 3 总结

综上所述，当我们调用方法时，首先进行方法的查找，如果查找失败，会进行方法的解析，此时`OC`会给我们一次对`sel`的处理机会，你可以在`resolveInstanceMethod:`（类方法对应`resolveClassMethod:`）中添加一个`IMP`；如果你没把握住这次机会，也就是解析失败时，会来到消息转发阶段，这个阶段有两个机会去处理`sel`，分别是快速转发的`forwardingTargetForSelector:`，以及慢速转发的`methodSignatureForSelector:`。当然，如果这些机会你都放弃了，那`OC`只好让程序崩溃。

下面用一副图总结方法的解析和转发流程

![](https://user-gold-cdn.xitu.io/2020/4/14/17178c89c46895dc?w=2696&h=1324&f=png&s=408302)

## 4 问题讨论

### 4.1 为什么引入消息转发机制？

在一个方法被调用之前，我们是没办法确定它的实现地址的，直到运行时，这个方法被调用的时候，我们才能真正知道它是否有实现，以及其具体的实现地址。这也就是所谓的“动态绑定”。

在编译期，如果编译器发现方法不存在，会直接报错；同样，在运行时，也有`doesNotRecognizeSelector`的处理。

在抛出`doesNotRecognizeSelector`这个异常信息之前，`OC`利用其动态绑定的特性，引入了消息转发机制，给予了我们额外的机会处理消息（解析 or 转发），这样的做法显然更加周全合理。

## 5 PS

* 源码工程已放到`github`上，请戳 [objc4-756.2源码](https://github.com/ConstantCody/objc4-756.2Demo)
* 你也可以自行下载 [苹果官方objc4源码](https://opensource.apple.com/tarballs/objc4/) 研究学习。
* 转载请注明出处！
