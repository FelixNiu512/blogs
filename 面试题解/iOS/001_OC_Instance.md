## OC对象的本质

以下问题都基于64bit环境

### 001、一个NSObject对象占用多少内存？

* 系统分配了16个字节给NSObject对象（通过`malloc_size`函数获得）
* 但是NSObject对象内部只使用了8个字节的空间（通过`class_getInstanceSize`函数获得）。

> 源码分析如下：  
> 
> * 关于`class_getInstanceSize`，实际上是获取类的成员变量占用的内存大小（指针大小的倍数，向上取整，其实就是内存对齐后的大小）；
* 关于`malloc_size`，其获取的是类的实例对象所占用的内存大小，即类在`alloc`时系统分配的内存大小；
	* `alloc`底层调用的是`_class_createInstanceFromZone`函数，该函数通过`cls->instanceSize`获取类的size，而`instanceSize`对size做了额外处理：如果`size < 16`，令`size = 16`。也就是说，类在`alloc`时，系统会为其分配至少`16个字节`的内存。

### 002、一个OC对象在内存中是如何布局的？

````
struct NSObject_IMPL {
	Class isa;
}

struct Persion_IMPL {
	struct NSObject_IMPL Student_IVARS;
	int _age;
}

struct Student_IMPL {
	struct Persion_IMPL Persion_IVARS;
	int _age;
}
````

* 内存的大端、小端模式  
* x/4xg：读取内存/4 16进制 8字节
* 内存对齐：结构体的最终内存大小，必须是最大成员大小的倍数

