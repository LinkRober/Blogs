+++
tags = ["Aspect","method swizzle","runtime"]
categories = ["ios"]
date = "2020-11-26T00:00:00Z"
title = "方法替换和Aspects"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++

 #### 带着问题看文章：

1.常规姿势的方法替换原理是什么

2.`Aspects`的方法替换原理是什么

3.为什么这样下面的代码这样hook之后，所有的实例的`viewWillAppear:`也被hook了

```
[[UIViewController class] aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionBefore usingBlock:^(){
        
} error:nil];
``` 

 4.为什么`Aspect`不能hook静态方法

 5.如果用先用Aspects hook了方法A，接着又用MethodSwizzle方法（下文有）对A进行了hook，两个hook都能执行吗？
<!--more-->

> ### 正文

#### `Runtime`
```
void MethodSwizzle(Class c,SEL origSEL,SEL overrideSEL)
   {
       Method origMethod = class_getInstanceMethod(c, origSEL);
       Method overrideMethod= class_getInstanceMethod(c, overrideSEL);
       if(class_addMethod(c, origSEL, method_getImplementation(overrideMethod),method_getTypeEncoding(overrideMethod)))
       {
            //当前类不存在`origSEL`
            class_replaceMethod(c,overrideSEL, method_getImplementation(origMethod), method_getTypeEncoding(origMethod));
       } else  {
            method_exchangeImplementations(origMethod,overrideMethod);
       }
}
```

##### 1. `Method class_getInstanceMethod(Class cls, SEL sel)`

从子类往父类递归查找Method

```
static method_t *
getMethod_nolock(Class cls, SEL sel)
{
    method_t *m = nil;

    runtimeLock.assertLocked();

    // fixme nil cls?
    // fixme nil sel?

    assert(cls->isRealized());

    while (cls  &&  ((m = getMethodNoSuper_nolock(cls, sel))) == nil) {
        cls = cls->superclass;
    }

    return m;
}
```

##### 2. `BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)`

为类添加新方法

如果当前类已经有这个`selector`，拿到该`selector`的`IMP`返回False；如果没有，或父类存在，会将这个方法的IMP指向新的IMP，返回True。

```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;

    mutex_locker_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);
}
```


```
static IMP 
addMethod(Class cls, SEL name, IMP imp, const char *types, bool replace)
{
    IMP result = nil;

    runtimeLock.assertLocked();

    checkIsKnownClass(cls);
    
    assert(types);
    assert(cls->isRealized());

    method_t *m;
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // already exists
        if (!replace) {
            result = m->imp;
        } else {
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
        // fixme optimize
        method_list_t *newlist;
        newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
        newlist->entsizeAndFlags = 
            (uint32_t)sizeof(method_t) | fixed_up_method_list;
        newlist->count = 1;
        newlist->first.name = name;
        newlist->first.types = strdupIfMutable(types);
        newlist->first.imp = imp;

        prepareMethodLists(cls, &newlist, 1, NO, NO);
        cls->data()->methods.attachLists(&newlist, 1);
        flushCaches(cls);

        result = nil;
    }

    return result;
}

```

#### 3. `IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)`

改变当前类中某个`selector`的`IMP`

```
IMP 
class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return nil;

    mutex_locker_t lock(runtimeLock);
    return addMethod(cls, name, imp, types ?: "", YES);
}
```

```
static IMP _method_setImplementation(Class cls, method_t *m, IMP imp)
{
    runtimeLock.assertLocked();

    if (!m) return nil;
    if (!imp) return nil;

    IMP old = m->imp;
    m->imp = imp;

    // Cache updates are slow if cls is nil (i.e. unknown)
    // RR/AWZ updates are slow if cls is nil (i.e. unknown)
    // fixme build list of classes whose Methods are known externally?

    flushCaches(cls);

    updateCustomRR_AWZ(cls, m);

    return old;
}
```

#### 4. `void method_exchangeImplementations(Method m1, Method m2)`

交换两个`Method`的`IMP`
```
void method_exchangeImplementations(Method m1, Method m2)
{
    if (!m1  ||  !m2) return;

    mutex_locker_t lock(runtimeLock);

    IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;


    // RR/AWZ updates are slow because class is unknown
    // Cache updates are slow because class is unknown
    // fixme build list of classes whose Methods are known externally?

    flushCaches(nil);

    updateCustomRR_AWZ(nil, m1);
    updateCustomRR_AWZ(nil, m2);
}

```

总结：</br>
上面包含了方法交互的两种方式：
- 已经存在的要交换的方法，直接交换方法的`IMP`指针
- 要交换的方法不存在，动态创建方法再交换方法的`IMP`

>Tips:交换`Method`
```
method_exchangeImplementations(class_getInstanceMethod(self, @selector(viewDidAppear:)),
                                   class_getInstanceMethod(self, @selector(ex_viewDidAppear:
```

#### `Aspects`

##### （一）消息转发
消息转发有3个阶段：
1. 当前对象动态添加方法来响应
2. 有备用对象能够响应该方法
3. 通过forwardInvocation方法来处理该方法

阶段3最灵活，和原来的类耦合最少。`Aspects`就是在这个阶段来做方法替换。它替换的是什么呢，就是`forwardInvocation`的`IMP`。

![](https://note.youdao.com/yws/api/personal/file/WEB95b6876b75aa452319fef5a997016d55?method=download&shareKey=809104bf2daedc6e202304bd2e45d2e8)
经过讨论应该是下面这样的流程
![](https://note.youdao.com/yws/api/personal/file/WEB84716c5dcfa90f82f31dca71e6baf268?method=download&shareKey=476d1e7dc88416af4d492f8d9e541243)

Aspects会强制将你要hook的方法的`IMP`指向`_objc_msgForward`的`IMP`，也就意味着你**直接**走到了消息转发的最后一步。接着在新的`IMP`： `__ASPECTS_ARE_BEING_CALLED__` 里面做各种切面的hook，before、instead、after。
```
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// Called after the original implementation (default)
    AspectPositionInstead = 1,            /// Will replace the original implementation.
    AspectPositionBefore  = 2,            /// Called before the original implementation.
    
    AspectOptionAutomaticRemoval = 1 << 3 /// Will remove the hook after the first execution.
};
```

##### （二）实例对象和类对象的Hook

类对象：<br>
直接将当前类对象`forwardInvocation`的`IMP`替换。

```
static void aspect_swizzleForwardInvocation(Class klass) {
    NSCParameterAssert(klass);
    // If there is no method, replace will act like class_addMethod.
    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
    ...
}
```

实例对象：</br>

![](https://note.youdao.com/yws/api/personal/file/WEBaab254928096c6f3850e8c5d74175e90?method=download&shareKey=f9ddd204eb6c299ad61feb797005dca4)

1.动态创建当前类的子类，2.再将子类的`forwardInvocation`进行替换，3.最后将类对象和原类`class`方法的`IMP`指向当前类的`class`方法的`IMP`,4.将当前类的`isa`指向动态创建的类
```
const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
Class subclass = objc_getClass(subclassName);
if (subclass == nil) {
		subclass = objc_allocateClassPair(baseClass, subclassName, 0);
		if (subclass == nil) {
            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
            return nil;
        }

		aspect_swizzleForwardInvocation(subclass);
		aspect_hookedGetClass(subclass, statedClass);
		aspect_hookedGetClass(object_getClass(subclass), statedClass);
		objc_registerClassPair(subclass);
	}
	//将self的isa指向动态创建的类
object_setClass(self, subclass);
```
这样做能保证现有的方法能正常执行；对当前类结构的改动最小。在`remove`的时候`isa`指针指回来就好了。
```
...
NSString *className = NSStringFromClass(klass);
        if ([className hasSuffix:AspectsSubclassSuffix]) {
            Class originalClass = NSClassFromString([className stringByReplacingOccurrencesOfString:AspectsSubclassSuffix withString:@""]);
            NSCAssert(originalClass != nil, @"Original class must exist");
            //将self的isa指向原来的类
            object_setClass(self, originalClass);
            AspectLog(@"Aspects: %@ has been restored.", NSStringFromClass(originalClass));
            ...
        }
...
```

##### （三）源码

###### 配置

```
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;
    
    ①aspect_performLocked(^{
        if (②aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            ③AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                ④[aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception.
                ⑤aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}

```
### ①</br>
对代码块加自旋锁
```
static void aspect_performLocked(dispatch_block_t block) {
    static OSSpinLock aspect_lock = OS_SPINLOCK_INIT;
    OSSpinLockLock(&aspect_lock);
    block();
    OSSpinLockUnlock(&aspect_lock);
}
```
### ②</br>
selector校验和tracker设置

1. 黑名单校验，这些方法是无法被hook的，有`retain`、`release`、`autorelease`、`forwardInvocation:`、
```
// Check against the blacklist.
    NSString *selectorName = NSStringFromSelector(selector);
    if ([disallowedSelectorList containsObject:selectorName]) {
        NSString *errorDescription = [NSString stringWithFormat:@"Selector %@ is blacklisted.", selectorName];
        AspectError(AspectErrorSelectorBlacklisted, errorDescription);
        return NO;
    }
```

2. `dealloc`无法在`AspectPositionBefore`情况下被hook；当前实例和类必须要都能响应要hook的方法
```
 AspectOptions position = options&AspectPositionFilter;
    if ([selectorName isEqualToString:@"dealloc"] && position != AspectPositionBefore) {
        NSString *errorDesc = @"AspectPositionBefore is the only valid position when hooking dealloc.";
        AspectError(AspectErrorSelectorDeallocPosition, errorDesc);
        return NO;
    }
if (![self respondsToSelector:selector] && ![self.class instancesRespondToSelector:selector]) {
        NSString *errorDesc = [NSString stringWithFormat:@"Unable to find selector -[%@ %@].", NSStringFromClass(self.class), selectorName];
        AspectError(AspectErrorDoesNotRespondToSelector, errorDesc);
        return NO;
    }
```
3. 如果是类对象，为该类及其继承体系上的所有类创建一个`AspectTracker`，并保存到一个全局单例的可变字典中。
```
        currentClass = klass;
        AspectTracker *parentTracker = nil;
        do {
            AspectTracker *tracker = swizzledClassesDict[currentClass];
            if (!tracker) {
                tracker = [[AspectTracker alloc] initWithTrackedClass:currentClass parent:parentTracker];
                swizzledClassesDict[(id<NSCopying>)currentClass] = tracker;
            }
            [tracker.selectorNames addObject:selectorName];
            // All superclasses get marked as having a subclass that is modified.
            parentTracker = tracker;
        }while ((currentClass = class_getSuperclass(currentClass)));
```
`AspectTracker结构`
![](https://note.youdao.com/yws/api/personal/file/WEB715024449547b00293407b3d99066c57?method=download&shareKey=d84e9dd7fe1949030441cea5e1156e96)

### ③</br>
通过runtime为当前类关联一个`AspectsContainer`类型的属性。命名方式是
```
//aspects__ + 方法名
eg:aspects__eat:andWater:
```
`AspectsContainer`保存了被hook方法的三种切面数组
![](https://note.youdao.com/yws/api/personal/file/WEB970471b0e4303ce3e0f095a4a2c92d93?method=download&shareKey=014892c4716701c7a5ed9026b33e33d1)

数组里面的对象`AspectIdentifier`包含了在进行所有切面操作所需的信息，原方法、切面block、切面block的签名、如何切以及原方法所属的类。
![](https://note.youdao.com/yws/api/personal/file/WEB78a33ecad4ae519b1113d730103d4def?method=download&shareKey=49e5bfc2a4f7297a29964987ea9fa873)

在创建`AspectIdentifier`对象的初始化方法里有两个校验逻辑，1. 方法签名的校验 2. 方法签名兼容性判断。

`Aspects`定义了一个结构和`NSGlobalBlock`类似的结构体`_AspectBlock`。我们的切面block(`NSGlobalBlock`)会被强转为`_AspectBlock`。
```
// Block internals.
typedef NS_OPTIONS(int, AspectBlockFlags) {
	AspectBlockFlagsHasCopyDisposeHelpers = (1 << 25),
	AspectBlockFlagsHasSignature          = (1 << 30)
};
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
Block方法签名的校验有两步：
1. `!(layout->flags & AspectBlockFlagsHasSignature)`(这应该是flags的一个规则)，如果`layout->flags`的值是0，返回nil
```
AspectBlockRef layout = (__bridge void *)block;
	if (!(layout->flags & AspectBlockFlagsHasSignature)) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't contain a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
}

```
2. 在`layout->descriptor`中是否存在`signature`。`desc`先向下偏移2个`unsigned long int`大小，如果存在copy和dispose函数继续向下偏移2个`unsigned long int`大小。这时候检查签名是否为空。
```
void *desc = layout->descriptor;
	desc += 2 * sizeof(unsigned long int);
	if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
		desc += 2 * sizeof(void *);
    }
	if (!desc) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't has a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
	const char *signature = (*(const char **)desc);
	return [NSMethodSignature signatureWithObjCTypes:signature];
```
![](https://lc-api-gold-cdn.xitu.io/3293f66aeb756e10eba7)


方法签名兼容性判断，通过对比方法签名来实现。
```
static BOOL aspect_isCompatibleBlockSignature(NSMethodSignature *blockSignature, id object, SEL selector, NSError **error) {
    NSCParameterAssert(blockSignature);
    NSCParameterAssert(object);
    NSCParameterAssert(selector);

    BOOL signaturesMatch = YES;
    NSMethodSignature *methodSignature = [[object class] instanceMethodSignatureForSelector:selector];
    if (blockSignature.numberOfArguments > methodSignature.numberOfArguments) {
        signaturesMatch = NO;
    }else {
        if (blockSignature.numberOfArguments > 1) {
            const char *blockType = [blockSignature getArgumentTypeAtIndex:1];
            if (blockType[0] != '@') {
                signaturesMatch = NO;
            }
        }
        // Argument 0 is self/block, argument 1 is SEL or id<AspectInfo>. We start comparing at argument 2.
        // The block can have less arguments than the method, that's ok.
        if (signaturesMatch) {
            for (NSUInteger idx = 2; idx < blockSignature.numberOfArguments; idx++) {
                const char *methodType = [methodSignature getArgumentTypeAtIndex:idx];
                const char *blockType = [blockSignature getArgumentTypeAtIndex:idx];
                // Only compare parameter, not the optional type data.
                if (!methodType || !blockType || methodType[0] != blockType[0]) {
                    signaturesMatch = NO; break;
                }
            }
        }
    }

    if (!signaturesMatch) {
        NSString *description = [NSString stringWithFormat:@"Blog signature %@ doesn't match %@.", blockSignature, methodSignature];
        AspectError(AspectErrorIncompatibleBlockSignature, description);
        return NO;
    }
    return YES;
}

```
如果方法签名个数不等返回NO；如果切面block的方法签名第1个参数（从第0个开始）的字符不是`@`返回NO；如果切面block的方法签名和要替换方法的方法签名从第2个开始不能一一匹配返回NO。

### ④</br>
将构建好的`AspectIdentifier`，添加到类对应的`AspectsContainer`中，调用Hook方法时要用。

### ⑤</br>
```
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```
这里就是调整`IMP`指针。下面这段代码主要有两个功能。
1. 将类的`forwardInvocation`的`IMP`指向自定义的`IMP`。这里分四种情况，（一）已经hook过的类，指实例对象，当前类的前缀是`_Aspects_`，直接返回。（二）类对象，上文已讨论过。（三）类对象和元类不是同一个类。（四）动态添加方法，上文也已讨论过。
```
//交换forwardInvocation IMP的代码
static NSString *const AspectsForwardInvocationSelectorName = @"__aspects_forwardInvocation:";
static void aspect_swizzleForwardInvocation(Class klass) {
    NSCParameterAssert(klass);
    // If there is no method, replace will act like class_addMethod.
    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
    if (originalImplementation) {
        class_addMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName), originalImplementation, "v@:@");
    }
    AspectLog(@"Aspects: %@ is now aspect aware.", NSStringFromClass(klass));
}
```
2.将要hook方法的`IMP`指向`_objc_msgForward`。


到这里完成了所有的配置工作

###### 运行时
直接上源码。</br>
先拿实例对象和类对象的`AspectsContainer`，可以为空。组装`AspectInfo`。先执行所有的`beforeAspects`切;接着如果存在`insteadAspects`，执行install切面，否则执行原方法；最后执行`afterAspects`切面。在每个切面执行的时候，实例对象和类对象的都要执行。如果有需要remove的切面都会加到`aspectsToRemove`这个数组里面，最后以remove掉。
```
#pragma mark - Aspect Invoke Point

// This is a macro so we get a cleaner stack trace.
#define aspect_invoke(aspects, info) \
for (AspectIdentifier *aspect in aspects) {\
    [aspect invokeWithInfo:info];\
    if (aspect.options & AspectOptionAutomaticRemoval) { \
        aspectsToRemove = [aspectsToRemove?:@[] arrayByAddingObject:aspect]; \
    } \
}

// This is the swizzled forwardInvocation: method.
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```
到这里基本差不多了，还有两个地方没讲，切面Block调用实现`[aspect invokeWithInfo:info]`和移除切面，有兴趣的可以看看。

最后附上一张整体流程图
![](https://note.youdao.com/yws/api/personal/file/WEB24a1a6c4e7c65ed963c9c65099042c69?method=download&shareKey=c4ea75a79f085b53d9d848ebd1b8543b)


##### （四）答题
1、2上面基本都已经说清楚了，我们从3看起：

3.为什么这样hook之后所有的实例方法也被hook了
```
[[UIViewController class] aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionBefore usingBlock:^(){
        
} error:nil];
```
答：这里考察的是`isa`的概念。实例对象调用方法的时候是在**isa指针指向的对象**里的方法列表去找，而**isa指针指向的对象**就是类对象，`Aspects`改变的是类对象方法列表里方法的`IMP`，所以无论那是实例调用了`viewWillAppear:`都会被hook

4.为什么`Aspect`不能hook静态方法
答：这里考察的也是`isa`的概念，可见`isa`在runtime中的地位非常重要。因为类对象调用方法的时候是到元类中去查找，而我们并没有对元类的方法进行hook。

5.如果用先用Aspects hook了方法A，接着又用MethodSwizzle对A进行了hook，两个hook都能执行吗？
答：先看这张图，以`viewDidAppear:`为列

![](https://note.youdao.com/yws/api/personal/file/WEBbd552a8d0fd08c81bddf036c20023229?method=download&shareKey=c8ab8006d377bd96f97cd3bc38ca9d1b)

`viewDidAppear:`的IMP先被指向了`forwardInvocatoin:`，当用MethodSwizzle进行第二次hook的时候，原来想指向`viewDidAppear:`正真的`IMP`的指针其实指向了`forwardInvocatoin:`，而`forwardInvocatoin:`被`Aspects`指向了自定义的`__ASPECTS_ARE_BEING_CALLED__`。这时候当调用`km_viewDidAppear:`的时候调用了`Aspects`的消息转发。在执行原方法的时候执行的是`aspects__t_viewDidAppear:`而不是当初动态添加的`aspects__viewDidAppear:`
```
[self doesNotRecognizeSelector:invocation.selector];
```
所以答案是用`MethodSwizzle`hook的方法可以执行。`Aspects`Hook的方法无法执行，已经无法拿到正确的`AspectsContainer`，动态添加的属性名称class_getSuperclass不对（是基于`viewDidAppear`创建的属性，而不是`km_viewDidAppear`）了，最后崩溃了。

但是`Aspects`还给了你最后一次机会如果被hook的类里面实现了方法`__aspects_forwardInvocation`还能挽救下崩溃的局面。
```
...
invocation.selector = originalSelector;
SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
if ([self respondsToSelector:originalForwardInvocationSEL]) {
    ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
}else {
    [self doesNotRecognizeSelector:invocation.selector];
}
...
```

以上的推断建立在当前类没有实现`viewDidAppear:`的前提下。如果当前类实现了`viewDidAppear:`MethodSwizzle拿到的就不是被交换过的IMP。这时候就可以形成闭环：`km_viewDidAppear:`->`viewDidAppear:(调用了[super viewDidAppear])`->`__ASPECTS_ARE_BEING_CALLED__`->`Aspects的hook方法`</br>
![](https://note.youdao.com/yws/api/personal/file/WEBb7863bbb0e16ab705af8d7443bfead85?method=download&shareKey=3b736c680f87c3c52f724c4845dca10b)


##### （四）知识点：
- 消息转发
- runtime的各种方法
1. `object_getClass(id _Nullable obj) `
2. `class_isMetaClass(Class _Nullable cls)`
3. `class_getSuperclass(Class _Nullable cls) `
4. `class_getInstanceMethod(Class _Nullable cls, SEL _Nonnull name)`
5. `class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, const char * _Nullable types) `
6. `method_getImplementation(Method _Nonnull m) `
7. `method_getTypeEncoding(Method _Nonnull m)`
8. `class_replaceMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, const char * _Nullable types)`
9. `method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2)`
10. `object_setClass(id _Nullable obj, Class _Nonnull cls)`
- isa指针
- block结构
- NSMethodSignature方法签名