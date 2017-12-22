+++
keywords = ["React Native"]
tags = ["ReactNative","JavsScript","ObjectC"]
categories = ["ReactNative"]
date = "2017-12-22T00:00:00Z"
title = "Native和JS通信概述"
draft = false
comments = true
showPagination = true
autoThumbnailImage = true

coverImage = '../../../rn_image.jpg'
showTags = true
showSocial = true
showDate = true

+++

本系列文章作为学习RN期间的总结

- [React Native如何集成到现有项目中](https://linkrober.github.io/bookshelf/2017/10/react-native%E5%A6%82%E4%BD%95%E9%9B%86%E6%88%90%E5%88%B0%E7%8E%B0%E6%9C%89%E9%A1%B9%E7%9B%AE%E4%B8%AD/)
- [React Native和Native间的通信实践](https://linkrober.github.io/bookshelf/2017/10/react-native%E5%92%8Cnative%E9%97%B4%E7%9A%84%E9%80%9A%E4%BF%A1/)
- [RCTRootView、RCTBridge做了什么](https://linkrober.github.io/bookshelf/2017/10/rctrootviewrctbridge%E5%81%9A%E4%BA%86%E4%BB%80%E4%B9%88/)
-  Native和JS通信概述

<!--more-->


#### 流程梳理
首先，无论是JS端还是OC端都会持有一个bridge负责对来自OC端或者JS端ModuleId、MedhodId的映射，再做消息处理。现附上一张来自JSPath作者Bang的图。
![来自JSPath作者Bang的一张流程图](../../../来自JSPath作者Bang的一张流程图.png)

JS调用某个方法向OC发送消息，方法首先会被分解成ModuleId、MethodId和参数，发送到OCBridge。那么OCBridge是如何接受的呢？

在第一次初始化ReactNative的时候模块信息会被初始化，包括模块名称、方法名。在注册模块的时候我们会用到`RCT_EXPORT_MODULE();`这个宏，宏展开
```
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }
```
方法1:如果自定义了模块的名称就使用自定义的名称而不是原类的名称

方法2:在+(void)load方法里对类进行一次注册，注册代码如下
```
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });

  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);

  // Register module
  [RCTModuleClasses addObject:moduleClass];
}
```
如果当前类遵循协议`RCTBridgeModule`,就把它放到一个静态变量数组`RCTModuleClasses`中。所以启动之后，所有RN需要的模块都会被动态载入。
这时候还有一个重要的东西需要处理就是callback，JSBridge会把callback缓存起来，仅仅把callbackId传递到OCBridge。
这时候OCBridge拿到了传过来的ModuleId、MethodId、callbackId，再通过OC端的配置表分别取出模块和方法。模块会被转化为`RCTModuleData`对象，方法都会被转化为一个`RCTModuleMethod`对象，看一下这个对象
```
{
  Class _moduleClass;
  const RCTMethodInfo *_methodInfo;
  NSString *_JSMethodName;

  SEL _selector;
  NSInvocation *_invocation;
  NSArray<RCTArgumentBlock> *_argumentBlocks;
}
```
发现了`_moduleClass`、`_argumentBlocks`、`_invocation`、`_selector`这时候就很容易想到消息转发的最后一个阶段，将消息的目标、selector、参数封装到NSInvocation中进行消息转发。

刚才上面讲到模块的注册，那么方法是如何筛选的呢？注意下面的3个宏
```
#define RCT_EXPORT_METHOD(method) \
  RCT_REMAP_METHOD(, method)
```
```
#define RCT_REMAP_METHOD(js_name, method) \
  _RCT_EXTERN_REMAP_METHOD(js_name, method, NO) \
  - (void)method;
```
```
#define _RCT_EXTERN_REMAP_METHOD(js_name, method, is_blocking_synchronous_method) \
  + (const RCTMethodInfo *)RCT_CONCAT(__rct_export__, RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    static RCTMethodInfo config = {#js_name, #method, is_blocking_synchronous_method}; \
    return &config; \
  }
```
在我们具体定义方法的时候并没有`-(void)`字符串前缀，因为在预编译的时候，宏会帮我们加上这段字符（看第二个宏）。
第三个宏主要实现了字符串的拼接，`(__LINE__, __COUNTER__)`的意思是为当前的宏生成一个唯一的tag（行数+一个数字），再和前面的`js_name`连接，最后再和`__rct_export__`连接，大概像这样

```
-(void)__rct_export__8976699MethodName{
	//
}
```
> 其实__COUNTER__是一个预定义的宏，这个值在编译过程中将从0开始计数，每次被调用时加1。

所有JSBridge传过来的参数都是`NSNumber`,OCBridge会把它们转化成基本类型和字符串。上面在提到JSBridge传参数的时候会带一个callbackId，这时候OCBridge会根据这个Id生成一个callback，保存在内存中。

这时候OC执行方法的调用，执行block。将参数和callbackId传到JSBridge，JSBridge根据Id找到之前缓存的JSCallback，进行调用。到这里就完成了个闭环。

> JS是怎么调用OC的呢？

其实并不是JS调用OC，而是JS将消息放到一个消息队列里面MessageQueue，等OC来取，这里面还有很多问题值得讨论多长时间取一次；不来取怎么办。有待我们继续研究

---

>为什么OC和JS可以实现这种框架？

要实现这个机制需要语言有动态反射的特性，即可以通过类/方法名字符串找到对应的类/方法进行调用，没有这特性就做不了。


#### 参考文档
[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)

[ReactNative iOS源码解析](http://awhisper.github.io/2016/06/24/ReactNative%E6%B5%81%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)



