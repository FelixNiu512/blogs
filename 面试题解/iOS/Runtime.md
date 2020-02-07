> 这是 **iOS高级面试题** 系列的 **Runtime篇**。欢迎大家阅读指正，也希望对大家有所帮助。整个系列如下：
> * [iOS之高级面试题总纲](https://github.com/ConstantCody/blogs/blob/master/面试题解/iOS/总纲.md)
> * [iOS高级面试题之Runtime篇](https://github.com/ConstantCody/blogs/blob/master/面试题解/iOS/Runtime.md)
> * 未完待续...

下面进入正题。

### 1. isKindOfClass 和 isMemberOfClass 的区别

1. **Q：** 请问下面代码的输出是什么？为什么？
````c
@interface Person : NSObject

@end

@implementation Person

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BOOL re1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
        BOOL re2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
        BOOL re3 = [(id)[Person class] isKindOfClass:[Person class]];
        BOOL re4 = [(id)[Person class] isMemberOfClass:[Person class]];
        NSLog(@"result : %hhd-%hhd-%hhd-%hhd", re1, re2, re3, re4);

        BOOL re5 = [(id)[NSObject alloc] isKindOfClass:[NSObject class]];
        BOOL re6 = [(id)[NSObject alloc] isMemberOfClass:[NSObject class]];
        BOOL re7 = [(id)[Person alloc] isKindOfClass:[Person class]];
        BOOL re8 = [(id)[Person alloc] isMemberOfClass:[Person class]];
        NSLog(@"result : %hhd-%hhd-%hhd-%hhd", re5, re6, re7, re8);
    }
    return 0;
}
````

2. **A：** 输出结果如下
````c
result : 1-0-0-0
result : 1-1-1-1
````

3. **题解：** 其实这道题总共考察了4个方法，分别是`isKindOfClass`类方法、对象方法，以及`isMemberOfClass`类方法、对象方法。破题的关键在于`isa指向图`。

* 首先分析两个类方法，其源码如下：
````c
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

// 上面两个类方法都调用了object_getClass()
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
````

先附上苹果官方的`isa指向图`

![](https://user-gold-cdn.xitu.io/2020/1/26/16fe0db0a22ebfba?w=1144&h=1170&f=png&s=240747)

现在探讨`obj->getIsa()`的结果：
> * 如果`obj`是一个对象，则返回该对象的所属类
> * 如果`obj`是一个类，则返回该类的元类
>   * 类与元类不一样
> * 如果`obj`是一个元类，则返回根元类

还有`tcls->superclass`的结果：
> * 如果`tcls`是类，则返回该类的父类
>   * `tcls`是根类的话（NSObject），它的父类是`nil`
> * 如果`tcls`是元类，则返回该元类的父类
>   * `tcls`是根元类的话，它的父类是`NSObject`类，父类的父类是`nil`

回归本题，先分析`+ (BOOL)isKindOfClass:(Class)cls`

对于`re1`，参数`cls`是`NSObject类`（根类），第一次`for`循环时，`tcls`是`NSObject元类`（根元类），然后往根元类的继承链上找，如果能找到根类，则返回`YES`。由于根元类的父类是根类，所以`re1`为`YES`，打印为`1`。

对于`re3`，参数`cls`是`Person类`，第一次`for`循环时，`tcls`是`Person元类`，然后往`Person元类`的继承链上找，如果能找到`Person类`，则返回`YES`。其先后对比了 `Person元类`、`根元类`、`根类`，没匹配到`Person类`，所以`re3`为`NO`，打印为`0`。

再分析`+ (BOOL)isMemberOfClass:(Class)cls`

对于`re2`，参数`cls`是`NSObject类`（根类），`object_getClass((id)self)`返回的也是`NSObject元类`（根元类），两者不相等，所以`re2`为`NO`，打印为`0`。

对于`re4`，参数`cls`是`Person类`，`object_getClass((id)self)`返回的也是`Person元类`，两者也不相等，所以`re4`为`NO`，打印为`0`。

因此，第一次打印的结果是
> result : 1-0-0-0

* 接着分析两个对象方法，其源码如下：
````c
- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
````
这两个对象方法比较简单，调用者`self`是对象，故`[self class]`是对象所属的类，而参数`cls`通常是另一个类，这两个方法不同的是：`- (BOOL)isMemberOfClass:(Class)cls`直接判断`[self class]`和`cls`是否一致，`- (BOOL)isKindOfClass:(Class)cls`则从`[self class]`开始，逐一匹配`[self class]`的继承链上的类。

对于`re5`、`re6`，`self`是`NSObject对象`，`[self class]`是`NSObject类`，`cls`是`NSObject类`，所以`re5`、`re6`都是`YES`；同理，`re7`、`re8`也是`YES`。

因此，第二次打印的结果是
> result : 1-1-1-1

4. 总结
* 一般地，对象方法`isKindOfClass`、`isMemberOfClass`的调用者是对象，比较的是对象所属的类与参数类是否一致，两者的不同之处在于
    * `isKindOfClass`会从对象的类开始匹配，直到根类；
    * `isMemberOfClass`仅仅判断对象的类是否等于参数类
* 对于类方法`isKindOfClass`、`isMemberOfClass`，两者的调用者通常是类，比较的是类的元类与参数类是否一致，不同之处在于
    * `isKindOfClass`匹配的是类的元类的继承链，最顶部是根类（因为根元类继承自根类）
    * `isMemberOfClass`仅仅判断调用者的元类是否等于参数类
> 对`isa`不清楚的同学推荐阅读 [OC源码分析之isa](https://juejin.im/post/5e0d4c686fb9a048401cff26)

