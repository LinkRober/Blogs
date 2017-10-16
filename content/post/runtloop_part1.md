+++
keywords = ["runtloop"]
tags = ["runloop"]
categories = ["Development"]
date = "2017-09-25T00:00:00Z"
title = "Run Loops (一)"
draft = false
+++

<!--more-->
>下面会用一些陌生的或者容易让人混淆的字符，我们先来统一概念再继续，这样能够让你更加愉快的阅读：

>**runloop:**iOS一个底层机制的专业术语。

>**run loop:**一种运行的循环。

>**Handler:**指`handlePort:`、`customSrc:`、`mySelector:`、`timerFired:`，指开发者希望当进入run loop的时候要执行的操作

>**run-loop observer：**观察runloop行为的观察者

>**run-loop mode：**每个runloop都要指定一种mode，一种mode可以对应多个input source

>**runloop object：**runloop相关的对象，这里分Cocoa和CoreFoundation中的


<i>Runloop</i>是线程架构相关的一部分，是事件处理的循环。你可以用它来安排循环执行任务，协调相关的事件（kCFRunLoopEntry、kCFRunLoopExit等）。
用几行示意代码来展示：
```
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
<i>Runloop</i>实际上是一个管理要处理的事件和消息的对象，并为开发者提供了一个入口来执行run loop的逻辑部分。一直处于类似于“接受消息-->等待-->处理”这样的一个循环汇总

><i>Runloop</i>的目的：一般我们自己写的线程在执行完相应的任务就会退出，而runloop**让你的线程在有事干的时候干活，没事干的时候休息，节省系统资源**。

<i>unloop</i>的管理不全是自动的，**对于自己的线程，必须要自己添加代码，并在适当的时候启动Run loop**来响应接受的事件。
Cocoa和Core Foundation提供了*runloop objects*来帮助你配置和管理线程的run loop。你的Application不需要自己创建这些对象(像alloc、init这些方法)。每个thread，包括Application的主线程，都已经有runloop object，有方法可以直接获取这这些对象，类似于这样的`[NSRunLoop currentRunLoop]`。
```
/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```
苹果提供了`CFRunLoopGetMain ` 、`CFRunLoopGetMain `这两个方法来创建<i>runloop</i>，如果它不被调用，线程中就不会有runloop。

>Cocoa中的`NSRunLoop`,和Core Foundation中的`CFRunLoopRef`等Run loop相关的类。它们的方法一般也是一一对应的。

唯一不同的是：你自己创建的线程需要明确的代码来让runloop跑起来，比如`[runLoop run]`这样的方法。但是对于主线程的<i>runloop</i>，系统会在App起来的时候自动设置并跑主线程里面的<i>runloop</i>，这是Application运行起来的一部分。


### Anatomy of Run Loop（剖析runloop）
“一个运行的循环”，看它名字的意思就能猜得到。它会循环不断的进入线程，在<i>runloop</i>进入之后做一些自己的事情。你的code提供了状态的控制用来实现<i>runloop</i>真正的循环部分。—— 换句话说，你的code提供`while`或`for`循环来驱动run loop。在你的循环中，你使用一个<i>runloop object</i>来运行事件处理的code，<i>runloop</i>接受事件并调用你已经准备好的方法。

```
/*
 这里doFireTimer方法就是你要处理的事件的code，<i>runloop</i>在从sleep状态被wakeup之后会执行改方法。
*/
[NSTimer scheduledTimerWithTimeInterval:0.016
                                     target:self
                                   selector:@selector(doFireTimer:)
                                   userInfo:nil
                                    repeats:YES];
```
上面我们提到**“你使用一个runloop object来运行事件处理的code”**，这里的事件是从哪里来呢？即事件源是什么？

事件源分两类：

- <i>Input source</i>所驱动的异步事件，一般它用来从一个线程向另一个线程发送事件或者从一个Application向另一个Application。
- <i>Timer sources</i>所驱动的同步事件，事件发生在一个已经预先设定好的时间，或者在重复的事件间隔发生。

当事件到达的时候，这两种source会按照系统指定的步骤来处理事件。


下图展示了一个<i>runloop</i>的概念结构和各种不同的sources。Input sources发送一个异步的事件来执行相对应的handler（例如，图中的`hanlePort`、`mySelector`等），并通过`runUntilDate:`方法（调用thread相关的[NSRunLoop](https://developer.apple.com/documentation/foundation/runloop)对象）来退出。Timer sources发送事件到runloop的handler，但不会引起runloop的退出。


![runloop-1.jpg](http://upload-images.jianshu.io/upload_images/1086250-2f8e40f8ce0f82c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

除了处理输入源对应的handler，<i>Runloop</i>生成关于run loop行为的通知。注册<i>run-loop</i> 的observer接受这些通知。使用它们在thread上做一些其他的操作。你可以在你的线程中使用Core Foundation里面的类来创建<i>run-loop observer</i>。

### Run Loop Modes

<i>run-loop mode</i>是被监听的<i>Input sources</i>和<i>Timers</i>的集合，同时还是被通知的<i>run-loop observer</i>的集合，大家可能比较疑惑怎么是两种东西的集合（source和obaserver），下面是我的理解，从代码出发：
`[runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];`
**线程的每个Source都要指定一个Mode，这里是NSDefaultRunLoopMode**
`CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopCommonModes);`
**观察者是观察该Runloop的某些Modes，kCFRunLoopCommonModes是mode的集合，每种mode都可以添加到Common中**


每次你运行你的<i>runloop</i>的时候，要指定一个具体的mode在<i>runloop</i>里面跑。在进入<i>runloop</i>的时候，只有和这个mode相关联的source能被对应的<i>run-loop observer</i>监听到，被允许向它们传递事件。而其他mode相关的source会一直等待，直到<i>run-loop mode</i>和source指定的mode相对应的时候才开始传递事件。

在你的代码中，你通过具体的名称来定义mode(`kCFRunLoopDefaultMode`、`kCFRunLoopCommonModes`等)。Cocoa和Core Foundation都定义了默认的mode和一些经常使用的mode。

你也能通过简单的字符串来自定义自己的mode。尽管custom mode定义起来好像很随意，但使用起来可不能随意啊。对于任何mode，你必须要确保添加一个或者更多的<i>sources</i>，<i>timers</i>或者<i>run-loop observer</i>，如果你没有做到，<i>run loop</i>就会直接退出。
```
//如果没有第二行代码，runloop会自动退出
NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
[runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
[runLoop run];

```
你可以使用mode过滤掉不想要的source。大多数情况下，你只需要用系统定义的“default”mode来运行你的runloop。

>Note:Mode是根据事件source来匹配的而不是根据事件的type。例如，你不可以使用鼠标的按下事件或者键盘按下事件来匹配mode。

下面是一些系统定义好的Modes

| Mode | Name |
|:----:|:----:|
| Default |[NSDefaultRunLoopMode](https://developer.apple.com/documentation/foundation/nsdefaultrunloopmode)(Cocoa)[kCFRunLoopDefaultMode](https://developer.apple.com/documentation/corefoundation/kcfrunloopdefaultmode)(Core Foundation)|
| Connection |[NSConnectionReplyMode](https://developer.apple.com/documentation/foundation/nsconnectionreplymode)(Cocoa)|
| Modal |[NSModalPanelRunLoopMode](https://developer.apple.com/documentation/appkit/nsmodalpanelrunloopmode)(Cocoa)|
|Event tracking|[NSEventTrackingRunLoopMode](https://developer.apple.com/documentation/foundation/runloopmode/1428765-eventtrackingrunloopmode)(Cocoa)|
|Common modes|[NSRunLoopCommonModes](https://developer.apple.com/documentation/foundation/runloopmode/1408609-commonmodes)(Cocoa)[kCFRunLoopCommonModes](https://developer.apple.com/documentation/corefoundation/kcfrunloopcommonmodes)(Core Foundation)|


### Input Sources
<i>Input source</i>通过异步的方式向你的线程传递事件。事件的源取决于<i>input source</i>的类型。<i>input source</i>大概分为两类：

- **Port-based port**
<i>Port-based port</i>监控你application的Mach port。

- **Custom input source** 
<i>Custom input source</i>监控自定义的事件source。

<i>runloop</i>不关心是哪种<i>input source</i>，这两种*input sources*系统都有实现。这两个<i>input source</i>唯一的不同就是它们如何发送信号。<i>Port-based port</i>通过内核自动发送信号。<i>Custom input source</i>只能从其他线程接受手动发送的信号。

当你创建一个<i>input source</i>，你把它赋值到<i>runloop</i>的mode。mode会影响被监控的input source。大多数情况，你在`NSDefaultRunLoopMode`下运行<i>runloop</i>，但是你也可以指定自定义的mode。如果一个<i>input source</i>不在当前被监控的mode中，它产生的一些事件将会被暂时的放在一边，直到下一个run loop的mode和input source的mode相同才开始处理这个input source产生的事件。

接下去的部分描述了一些input source

##### Port-Based Sources
Cocoa和Core Foundation通过提供端口相关的对象和方法来创建<i>Port-Based Source</i>。例如，在Cocoa中，你没必要直接创建input source。你使用`NSPort`的方法`[NSPort port]`，添加port到run loop中。Port object会为你创建和配置所需的<i>Port-Based Sources</i>。

在 Core Foundation中，你必须要自己创建port和它的run loop source。用 [CFMachPortRef](https://developer.apple.com/documentation/corefoundation/cfmachport), [CFMessagePortRef](https://developer.apple.com/documentation/corefoundation/cfmessageportref),  [CFSocketRef](https://developer.apple.com/documentation/corefoundation/cfsockeref)这些不透明指针创建相应的对象。

如何配置和设置自定义的<i>Port-Based Sources</i>，请看[ Configuring a Port-Based Input Source](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-131281)

##### Custom Input Sources（Core Foundation）
为了创建自定义的input source，你必须使用Core Foundation中的[CFRunLoopSourceRef](https://developer.apple.com/documentation/corefoundation/cfrunloopsource)不透明类相关的方法。你需要在回调方法中配置这个Custom Input Sources。Core Foundation会在配置Custom Input Sources、处理输入事件和当source要从runloop中移除的时候调用这几个回调。

对于如何创建一个custom input source请看这里[Defining a Custom Input Source](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW3)

##### Cocoa Perform Selector Sources（Cocoa）
除了<i>Port-Based Sources</i>，Cocoa定义了一个<i>custom input source</i>，这允许你向任何线程中发送选择子。和<i>Port-Based Sources</i>一样，执行选择子selector的请求在同一个线程上是连续的，当多个方法同时向线程发送请求的时候可能会导致同步的问题。和<i>Port-Based Sources</i>不同的是：一个<i>perform selector source</i>执行完它的selector，它自己会从runloop中移除。

当向另一个线程发送selector请求时，目标线程必须有一个active的<i>runloop</i>。这意味着你必须等待，直到该线程中runloop的状态变成 active。因为主线程会自己开始一个<i>runloop</i>，只要系统一调用Application delegate的`applicationDidFinishLaunching:`（这时候系统的runloop就创建出来了），你就可以向主线程发送selector请求。每次<i>Runloop</i>会处理所有队列的selector请求，而不是只处理一个队列的请求。下面列出了一些perform方法

|Methods|
|:---:|
|[performSelectorOnMainThread:withObject:waitUntilDone:](https://developer.apple.com/documentation/objectivec/nsobject/1414900-performselector)、[performSelectorOnMainThread:withObject:waitUntilDone:modes:](https://developer.apple.com/documentation/objectivec/nsobject/1411637-performselector)|
|[performSelector:onThread:withObject:waitUntilDone:](https://developer.apple.com/documentation/objectivec/nsobject/1414476-perform)、[performSelector:onThread:withObject:waitUntilDone:modes:](https://developer.apple.com/documentation/objectivec/nsobject/1417922-perform)|
|[performSelector:withObject:afterDelay:](https://developer.apple.com/documentation/objectivec/nsobject/1416176-perform)、[performSelector:withObject:afterDelay:inModes:](https://developer.apple.com/documentation/objectivec/nsobject/1415652-perform)|
|[cancelPreviousPerformRequestsWithTarget:](https://developer.apple.com/documentation/objectivec/nsobject/1417611-cancelpreviousperformrequests)、[cancelPreviousPerformRequestsWithTarget:selector:object:](https://developer.apple.com/documentation/objectivec/nsobject/1410849-cancelpreviousperformrequests)|

##### Timer Sources
<i>Timer sources</i>会在将来的某个时间，发送同步事件到你的当前的线程。Timer是线程用来通知它自己的一种方式。
尽管它是基于时间的通知，一个timer并不能保证在准确的时间点执行事件。和其他的<i>Input source</i>一样，Timer和你<i>run loop</i>指定的mode有关系。如果Timer的mode不是你当前runloop正在跑的mode，它是不会被触发的，直到你的<i>runloop</i>运行的的mode是你的Timer所支持的。类似的，如果一个Timer被触发，但是当runloop在执行Handler，Timer将会一直等待，直到下次再次进入runloop的时候才会执行。如果这个<i>runloop</i>不再跑了，这个Timer将永远不会被触发。

你可以配置Timer来运行事件，一次或者不停重复。一个重复的Timer会基于设定好的触发时间，自动重新安排它自己到run loop中，但这个时间间隔并不一定准确。例如，如果一个Timer在一个特定时间被触发，每5秒重复一次。即使实际的触发时间被延迟了，被安排的触发时间总落在5秒的延迟时间间隔之内。如果触发时间被延迟太多，会导致它错过了一次或者更多被安排到run loop的机会。在一个不恰当的时机被触发之后，Timer则会被安排到下一次run loop。
关于配置timer sources的更多信息，请看[Configuring Timer Sources](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW6)。更多相关信息[NSTimer Class Reference](https://developer.apple.com/documentation/foundation/timer)或[CFRunLoopTimer Reference](https://developer.apple.com/documentation/corefoundation/cfrunlooptimer-rhk)。

##### Run Loop Observers
一般的source是在一个异步或者同步的事件发生的时候被触发的，而<i>run-loop observer</i>是在<i>runloop</i>它执行的时候触发的。你可以通过<i>run-loop observer</i>来准备一些线程待处理的事件，或者在线程sleep之前，在线程中做一些事。<i>run-loop observers</i>可以观察到下面的一些事件：

* 进入<i>runloop</i>的时候
* 当<i>runloop</i>将要处理timer的时候
* 当<i>runloop</i>将要处理input source的时候
* 当<i>runloop</i>将要sleep的时候
* 当<i>runloop</i>已经醒来，还没有开始处理event的时候
* 当<i>runloop</i>退出的时候

你可以使用Core Foundation向Application添加<i>run-loop observer</i>。为了创建一个<i>run_loop observer</i>，你需要使用[CFRunLoopObserverRef](https://developer.apple.com/documentation/corefoundation/cfrunloopobserver)类。这个类会追踪你自定义callback函数、你感兴趣的`CFRunLoopActivity`。

和Timer相似，<i>run-loop observer</i>可以只运行一次，或者不停重复。只运行一次的<i>run-loop observer</i>在一次run loop之后就会从<i>runloop</i>中移除。当你创建<i>run-loop observer</i>的时候，你要决定r是一次还是不停重复。

对于如何创建一个<i>run-loop observer</i>的例子，看[ Configuring the Run Loop](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW18),更多相关信息[CFRunLoopObserver Reference](https://developer.apple.com/documentation/corefoundation/cfrunloopobserver-ri3)

##### The Run Loop Sequence of Events
每次你运行<i>runloop</i>，你线程的<i>runloop</i>会处理之前挂起的事件，并且向*run-loop observer*发送通知。<i>runloop</i>执行的顺序如下面所列出的：

1. 通知<i>run-loop observer</i>，进入<i>runloop</i>
2. 通知<i>run-loop observer</i>，已经准备好的Timer将要触发
3. 通知<i>run-loop observer</i>，以及准备好的*no-port-based input source*将要被触发
4. 触发<i>no-port-based input source</i>
5. 如果一个<i>port-based input source</i>准备好等待触发，立即处理该事件，再从第九步开始执行
6. 通知<i>run-loop observer</i>，线程将要sleep
7. 线程sleep直到下面的事件发生：
	*	<i>port-based input source</i>事件到达
	* 	一个Timer触发了
	*  <i>runloop</i>超时了
	*  <i>runloop</i>被手动唤醒
8. 通知<i>run-loop observer</i>，线程将要被唤醒
9. 处理挂起的事件
	* 	如果一个用户定义的timer触发了，处理相关事件，重新开始循环，回到第二步继续执行
	*  如果<i>input source</i>触发，传递相应的事件
	*  如果<i>runloop*</i>明确的被唤醒，但是还没有超时，重新开始循环，回到第二步继续执行

10. 通知<i>run_loop observer</i>，<i>runloop</i>已经退出

>**Tip:**no-port-based input source就是我们常说的**source0**，port-based input source就是**source1**

因为Timer和input source的通知是在事件被触发之前，所以这之间有一个过渡时间。如果你要在这个时间做一些事情，你可以使用**sleep**和 **awake-from-sleep** 通知来帮助你获得这段时间的控制权。

一个<i>runloop*</i>可以通过使用<i>run loop object</i>被唤醒。其他的事件也可能引起<i>runloop*</i>被唤醒。例如，添加<i>no-port-based input source</i>来唤醒<i>runloop*</i>以至于input source可以被立即处理，而不是等待其他的事件发生。
***

>iOS系统中有很多用到了RunLoop的地方：

>- AutoreleasePool：监听`kCFRunLoopEntry`事件，保证在所以回调之前创建自动释放池；监听`BeforeWaiting `事件，释放旧的线程池，创建新的线程池；监听`kCFRunLoopExit`事件，保证线程池最后被释放。
>- 事件响应：通过port-based input source监听用户的事件，并转发给App进程，如用UIControl的事件。
>- 手势识别：监听`kCFRunLoopBeforeWaiting`事件在事件的回调函数中标记待处理的手势，并处理手势。
>- 界面更新：`setNeedsLayout`、`setNeedsDisplay`都会将该view标记为待处理，等到下一个runloop的时候会布局UI。
>- 定时器：就是一个CFRunLoopTimerRef
>- PerformSelecter：将方法扔到某个线程中，在下一个run loop的时候执行。
>- 关于GCD：dispatch_async(dispatch_get_main_queue(), block)向主线程的run loop发送消息，唤醒run loop。并执行block中的内容。

PPT总结一下：

![Screen Shot 2017-09-05 at 10.12.16 AM.png](http://upload-images.jianshu.io/upload_images/1086250-cb2c4b4a6d0973e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Screen Shot 2017-09-05 at 10.12.37 AM.png](http://upload-images.jianshu.io/upload_images/1086250-fb6430f46efe4735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Screen Shot 2017-09-05 at 10.12.53 AM.png](http://upload-images.jianshu.io/upload_images/1086250-7118fdeabb5ffe76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Screen Shot 2017-09-05 at 10.13.01 AM.png](http://upload-images.jianshu.io/upload_images/1086250-e7ae08b4d61ec9e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Screen Shot 2017-09-05 at 10.12.47 AM.png](http://upload-images.jianshu.io/upload_images/1086250-073579ed39612c1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Screen Shot 2017-09-05 at 10.13.12 AM.png](http://upload-images.jianshu.io/upload_images/1086250-cae92bc60b877b48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Screen Shot 2017-09-05 at 10.13.21 AM.png](http://upload-images.jianshu.io/upload_images/1086250-fae8de2bdb8232aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![Screen Shot 2017-09-05 at 10.13.27 AM.png](http://upload-images.jianshu.io/upload_images/1086250-ac972598a07a90bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)