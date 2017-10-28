+++
keywords = ["React Native"]
tags = ["React Native"]
categories = ["development"]
date = "2017-10-28T00:00:00Z"
title = "RCTRootView、RCTBridge做了什么"
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
- [React Native和Native间的通信](https://linkrober.github.io/bookshelf/2017/10/react-native%E5%92%8Cnative%E9%97%B4%E7%9A%84%E9%80%9A%E4%BF%A1/)
- RCTRootView、RCTBridge做了什么

<!--more-->

本章主要讨论，RCTRootView、RCTBridge彼此之间的关系，以及它们做了什么。先放上类图


![相关类图](../../../相关类图.png)

#### RCTRootView
它是RN在Native上展示的容器，`RCTJavaScriptWillStartLoadingNotification`、`RCTJavaScriptDidLoadNotification`、`RCTContentDidAppearNotification`这些通知注册都在里面。同时也是运行js的入口。

{{< hl-text blue >}}RCTJavaScriptWillStartLoadingNotification{{< /hl-text >}}</br>
当你在Debug模式下，选择**Live Reload**每次保存js文件的时候就会重新加载js。之前的`reactTag`会清空，等待重新赋值，但第一次初始化并不会触发这个通知。因为通知这时候还没注册。

{{< hl-text blue >}}RCTJavaScriptDidLoadNotification{{< /hl-text >}}</br>
同样的在**Live Reload**模式下，每次保存，才会正真的重新加载js，完毕之后，开始视图view的重新初始化即，`RCTRootContentView`，然后`RCTRootView`会通过bridge发送消息

```
- (void)enqueueJSCall:(NSString *)module 
			   method:(NSString *)method 
			     args:(NSArray *)args 
		   completion:(dispatch_block_t)completion;
```
它把模块名称，方法名和在native中定义的属性（上一节提到的properies）发送到js端，让js端完整的逻辑渲染

>它们是如何控制在第一次初始化rootview的时候不执行`RCTJavaScriptDidLoadNotification`这个通知定义的事件的呢？是通过下面的判断

```
 RCTBridge *bridge = notification.userInfo[@"bridge"];
  if (bridge != _contentView.bridge) {
    [self bundleFinishedLoading:bridge];
  }
```
{{< alert danger >}}
`RCTBatchedBridge.mm`是`RCTBridge.m`的子类，但是在后面可能会废弃。`RCTBridge`通过一种向下转型的方式，让其拥有了子类的能力。
{{< /alert >}}


{{< hl-text blue >}}RCTContentDidAppearNotification{{< /hl-text >}}</br>
在视图出来的时候RN会通过这个通知隐藏loading页面，loading页面也是可以自定义的，通过重新它的`setLoadingView:`方法。

{{< hl-text yellow >}}总结：{{< /hl-text >}}

- 负责RN和Native通信的`RCTBridge`实例的初始化。
- 正真用来展示视图的`RCTRootContentView`的初始化，完事后通过` [self insertSubview:_contentView atIndex:0];`的方式加到rootview上。
- 负责loading视图的展示和隐藏。
- 负责在刷新视图时清空reactTag，它是也是区分js中每个view的identifier。
- 负责通知`RCTBridge`在合适的时候向JS发送已经准备好的信息，模块名，方法名、参数一起传递过去。


#### RCTBridge

所有的模块都保存在内存的静态常量区域，这个内存空间是在`RCTBridge`中被初始化的。因此这些模块属于全局的变量，可以在不同的bridge之间共享。可以通过`NSString *RCTBridgeModuleNameForClass(Class cls)`拿到每个模块的名称。

```
NSString *RCTBridgeModuleNameForClass(Class cls)
{
#if RCT_DEBUG
  RCTAssert([cls conformsToProtocol:@protocol(RCTBridgeModule)],
            @"Bridge module `%@` does not conform to RCTBridgeModule", cls);
#endif

  NSString *name = [cls moduleName];
  if (name.length == 0) {
    name = NSStringFromClass(cls);
  }

  if ([name hasPrefix:@"RK"]) {
    name = [name substringFromIndex:2];
  } else if ([name hasPrefix:@"RCT"]) {
    name = [name substringFromIndex:3];
  }

  return name;
}
```
从上面的代码可以看出，在native中定义的模块在被传递到js的时候会加上`RK`或者`RCT`前缀。

{{< alert info no-icon >}}
模块都是遵循`RCTBridgeModule`协议的，用宏`RCT_EXPORT_MODULE();`进行声明；
{{< /alert >}}

JS队列的初始化会在`RCTBridge`被调用到的时候通过`+ (void)initialize`进行初始化，而且它是一个单列对象。通过C的`extern __attribute__((visibility("default")))`暴露出来，让该队列变量暴露在动态链接库之外

```
dispatch_queue_t RCTJSThread;

+ (void)initialize
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{

    // Set up JS thread
    RCTJSThread = (id)kCFNull;
  });
}
```
每一个当前视图正在使用的bridge也会放在静态常量区，每次视图切换的时候都会重新赋值。

```
static RCTBridge *RCTCurrentBridgeInstance = nil;

/**
 * The last current active bridge instance. This is set automatically whenever
 * the bridge is accessed. It can be useful for static functions or singletons
 * that need to access the bridge for purposes such as logging, but should not
 * be relied upon to return any particular instance, due to race conditions.
 */
+ (instancetype)currentBridge
{
  return RCTCurrentBridgeInstance;
}

+ (void)setCurrentBridge:(RCTBridge *)currentBridge
{
  RCTCurrentBridgeInstance = currentBridge;
}
```
在`RCTBridge`的初始化方法中同时还会对`RCTBatchedBridge`进行初始化，由其发送消息开始加载js。其中分为两步：

- 初始化modul和load资源
- 设置JS的Executor和modul的配置

它们在同一个队列`dispatch_queue_t bridgeQueue = dispatch_queue_create("com.facebook.react.RCTBridgeQueue", DISPATCH_QUEUE_CONCURRENT);`里的两个不同的group中异步执行：

- `dispatch_group_t initModulesAndLoadSource = dispatch_group_create();`
- `dispatch_group_t setupJSExecutorAndModuleConfig = dispatch_group_create();`

因为在第一个group中的异步方法中也存在异步操作所有作者使用的` dispatch_group_enter()` 和 `dispatch_group_leave()`。

第二个group直接用的`dispatch_group_async(dispatch_group_t  _Nonnull group, dispatch_queue_t  _Nonnull queue, ^(void)block)`
当第二group的异步任务完成之后（`initModulesAndLoadSource`依赖`setupJSExecutorAndModuleConfig`），就开始执行对应的notify闭包向js注入json字符串

```
dispatch_group_notify(setupJSExecutorAndModuleConfig, bridgeQueue, ^{
      // We're not waiting for this to complete to leave dispatch group, since
      // injectJSONConfiguration and executeSourceCode will schedule operations
      // on the same queue anyway.
      [performanceLogger markStartForTag:RCTPLNativeModuleInjectConfig];
      [weakSelf injectJSONConfiguration:config onComplete:^(NSError *error) {
        [performanceLogger markStopForTag:RCTPLNativeModuleInjectConfig];
        if (error) {
          RCTLogWarn(@"Failed to inject config: %@", error);
          dispatch_async(dispatch_get_main_queue(), ^{
            [weakSelf stopLoadingWithError:error];
          });
        }
      }];
```
待第二个group结束，开始执行第一个group对应的notify闭包,开始加载js，加载结束之后发出`RCTJavaScriptDidLoadNotification`通知。

{{< alert info no-icon >}}
执行JS所有相关的内容都是`@protocol RCTJavaScriptExecutor`这个协议负责，定义在bridge中`@property (nonatomic, weak, readonly) id<RCTJavaScriptExecutor> javaScriptExecutor;`
{{< /alert >}}

当你在Debug模式下save或者退出当前页面的时候收到reload命令，该类会在主线程中将当前的bridge置空，如果没有成功会调用其所遵循的协议`@protocol RCTInvalidating`中的`invalidate`方法进行一系列的局部变量释放操作

```
- (void)invalidate
{
  RCTBridge *batchedBridge = self.batchedBridge;
  self.batchedBridge = nil;

  if (batchedBridge) {
    RCTExecuteOnMainQueue(^{
      [batchedBridge invalidate];
    });
  }
}
```

{{< hl-text yellow >}}总结：{{< /hl-text >}}

- 保存模块的集合 --`RCTModuleClasses` 
- 定义了JS队列 -- `dispatch_queue_t RCTJSThread`
- 定义当前正在使用的bridge，并负责切换
- 负责`RCTBatchedBridge`的初始化，调用子类bridge，开始执行加载js的一系列操作
- 负责在view消失或者重新加载的时候释放之前的一些对象（eg. currentbridge,遵循`RCTBridgeModule`协议的对象）。

















