+++
title = "iOS中的方法编码"
date = "2021-06-08T00:00:00Z"
categories = ["iOS"]
keywords = ["runtime","encode"]
tags = ["runtime","encode"]
draft = false
+++

当我们在日常开发中，如果涉及到一些iOS底层的东西就可能会遇到方法编码，这里了解这门语言必不可少的一个环节。

<!--more-->

##### Object-C 编码类型

|类型|意义|
|:-:|:-:|
|c|A char|
|i|An int|
|s|A short|
|l|A long l is treated as a 32-bit quantity on 64-bit programs|
|q| A long long|
|C|An unsigned char|
|I|An unsigned int|
|S|An unsigned short|
|L|An unsigned long|
|Q|An unsigned long long|
|f|A float|
|d|A double|
|B|A C++ bool or a C99 _Bool|
|v|A void|
|*|A character string(char *)|
|@|An object (whether statically typed or typed id)|
|#|A class object(Class)|
|:|A method selector(SEL)|
|[array type]|An array|
|{name=type...}|A structure|
|(name=type...)|A union|
|bnum|A bit field of num bits|
|^type|A pointer to type|
|?|An unknown type(among other things, this code is used for function pointers)|

>[来自苹果文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

如何获取某个类型的编码：
```
NSLog(@"int        : %s", @encode(int));
NSLog(@"float      : %s", @encode(float));
NSLog(@"float *    : %s", @encode(float*));
NSLog(@"char       : %s", @encode(char));
NSLog(@"char *     : %s", @encode(char *));
NSLog(@"BOOL       : %s", @encode(BOOL));
NSLog(@"void       : %s", @encode(void));
NSLog(@"void *     : %s", @encode(void *));

NSLog(@"NSObject * : %s", @encode(NSObject *));
NSLog(@"NSObject   : %s", @encode(NSObject));
NSLog(@"[NSObject] : %s", @encode(typeof([NSObject class])));
NSLog(@"NSError ** : %s", @encode(typeof(NSError **)));

int intArray[5] = {1, 2, 3, 4, 5};
NSLog(@"int[]      : %s", @encode(typeof(intArray)));

float floatArray[3] = {0.1f, 0.2f, 0.3f};
NSLog(@"float[]    : %s", @encode(typeof(floatArray)));

typedef struct _struct {
    short a;
    long long b;
    unsigned long long c;
} Struct;
NSLog(@"struct     : %s", @encode(typeof(Struct)));
```


##### 实例或类方法编码

###### 定义一个Person类
```
@interface Person : NSObject

- (void)say:(NSString *)something;

@end


@implementation Person

- (void)say:(NSString *)something {
    NSLog(@"Well Done");
}

```

###### 获取Person实例方法的方法编码
```
Person *p = [Person new];
Method method = class_getInstanceMethod([Person class], @selector(say:));
const char *paramsType = method_getTypeEncoding(method);
//输出："v24@0:8@16"
```
>类方法使用`class_getClassMethod(<Class  _Nullable __unsafe_unretained cls>, <SEL  _Nonnull name>)`获取Method

`v`返回值是`void`类型，`24`方法参数占24个字节，`@`指self，`0`代表self起始地址的偏移量是0，`:`指selector，`8`代表selector起始地址偏移量是8，`@`指入参是一个对象，`16`代表入参起始地址的偏移量是16。16+8(对象占8个字节大小)=24，就是这个方法的大小了。

##### 代码块
苹果的Block内存布局：
```
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30)  // compiler
};

// revised new layout

#define BLOCK_DESCRIPTOR_1 1
struct Block_descriptor_1 {
    unsigned long int reserved;
    unsigned long int size;
};

#define BLOCK_DESCRIPTOR_2 1
struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};

#define BLOCK_DESCRIPTOR_3 1
struct Block_descriptor_3 {
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;
    const char *layout;
};

struct Block_layout {
    void *isa;
    volatile int flags; // contains ref count
    int reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
```
可以看到这里有三个结构体`Block_layout` `Block_descriptor_1` `Block_descriptor_2` `Block_descriptor_3`主结构体是`Block_layout`，`Block_descriptor_1`是每个Block结构体都有的，`Block_descriptor_2` `Block_descriptor_3`要根据`flags`来判断。如果`flags&BLOCK_HAS_COPY_DISPOSE`为真，就会有`Block_descriptor_2`结构体；如果`flags&BLOCK_HAS_SIGNATURE`为真，就会有`Block_descriptor_3`结构体。一个完全体的Block结构体就是:
```
struct Block_layout {
    void *isa;
    volatile int flags; // contains ref count
    int reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    struct Block_descriptor_2 *descriptor2;
    struct Block_descriptor_3 *descriptor3;
};
```
`Block_descriptor_3`里面就保存了我们想要的**方法编码**。将Block的结构体铺开，大概是下面的样子（一张Aspect实现的结构体的内部布局图）
![](https://lc-api-gold-cdn.xitu.io/3293f66aeb756e10eba7)
这时候我们可以先定义一个Block结构`AspectBlockRef`，通过`__bridge void *`将OC对象转换成结构体指针。
Aspect定义的结构体
```
typedef struct _AspectBlock {
    __unused Class isa;
    AspectBlockFlags flags;
    __unused int reserved;
    void (__unused *invoke)(struct _AspectBlock *block, ...);
    struct {
        unsigned long int reserved;
        unsigned long int size;
        // requires AspectBlockFlagsHasCopyDisposeHelpers
        void (*copy)(void *dst, const void *src);
        void (*dispose)(const void *);
        // requires AspectBlockFlagsHasSignature
        const char *signature;
        const char *layout;
    } *descriptor;
    // imported variables
} *AspectBlockRef;
```
成功转换得到结构体之后，可以访问成员`descriptor`，再通过指针的偏移拿到签名`signature`所在的地址。

```
static NSMethodSignature *aspect_blockMethodSignature(id block, NSError **error) {
    AspectBlockRef layout = (__bridge void *)block;
    if (!(layout->flags & AspectBlockFlagsHasSignature)) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't contain a type signature.", block];
//        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
    void *desc = layout->descriptor;
    desc += 2 * sizeof(unsigned long int);
    if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
        desc += 2 * sizeof(void *);
    }
    if (!desc) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't has a type signature.", block];
//        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
    const char *signature = (*(const char **)desc);
    return [NSMethodSignature signatureWithObjCTypes:signature];
}
```
当拿到签名`NSMethodSignature`之后我们可以通过runtime获得我们所需要的方法编码:
```
NSString *getMethodSignatureTypeEncoding(NSMethodSignature *methodSignature){
    NSMutableString *str = @"".mutableCopy;
    const char *rtvType = methodSignature.methodReturnType;
    [str appendString:[NSString stringWithUTF8String:rtvType]];

    for (int i = 0; i < methodSignature.numberOfArguments; i ++) {
        const char *type = [methodSignature getArgumentTypeAtIndex:i];
        [str appendString:[NSString stringWithUTF8String:type]];
    }
    return [str copy];
}
```
写个例子：
```
typedef id (^TestBlock)(id a);
TestBlock block = ^id (id a){
        return [NSObject new];
    };
NSMethodSignature *mallocBlockSignature = aspect_blockMethodSignature(block, NULL);
NSString *methodTypes = getMethodSignatureTypeEncoding(mallocBlockSignature);
//输出：@@?@
```
第一个`@`是返回值；第二个`@`是self；`?`这是Block和普通的方法编码有别的地方，不是`:`而是`?`；第三个`@`是入参。

##### 为什么要研究方法编码
我们在做一些框架的时候并不能像做业务开发一样直接向这个对象发送消息，我们需要自己去组装，通过`NSInvocation`或`消息转发`进行发送。比如，`Aspect` `一些组件间通信框架` `JSPatch`等。


##### 参考资料

[Block_private.h](https://opensource.apple.com/source/libclosure/libclosure-59/Block_private.h.auto.html)

[Objective C, Encoding and You
](https://dmaclach.medium.com/objective-c-encoding-and-you-866624cc02de)