+++
date = "2017-09-26T00:00:00Z"
title = "Run Loops (二)"
keywords = ["runloop"]
tags = ["runloop","ObjectC"]
categories = ["iOS"]

+++

 
### When Would You Use a Run Loop?
你唯一要使用run loop，就是当你要在application中创建线程的时候。你Application的主线程是架构很重要的一部分。所以，iOS系统为app的提供了<i>runloop</i>代码，并自动开始。在iOS中运行Main loop作为app启动步骤的一部分。
<!--more-->

对于其他的线程，你需要考虑是否必须要使用<i>run loop</i>。如果有需要，配置并自己启动它。你不需要在每个线程中都启用run loop。例如，如果你使用一个线程来执行一些很长时间的运行和已经准备好的任务，你可能要避免使用它。Run loop的目的主要是用来解决线程间的通信。例如，如果你打算做下面的事情，你需要开启一个run loop。

* 使用port或者自定义input source来和其他线程通讯
* 在线程中使用timer
* 在Cocoa application使用`performSelector… `方法
* 维持线程来执行周期性的任务

如果你选择使用run loop，配置和设置就是接下来要做的。纵观整个线程编码过程，你应该有一个明确的认识在什么时候退出线程，这比强制退出好很多。关于如何配置和退出run loop的信息在这[Using Run Loop Objects](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW5)。

### Using Run Loop Objects
<i>run loop</i>对象提供了主要的接口来添加input sources，timers，run-loop observers到<i>run loop</i>中再运行它。每个线程有一个和它相关的run loop object。在Cocoa中，这个对象是[NSRunLoop](https://developer.apple.com/documentation/foundation/runloop)的实例。在应用的底层，它是[CFRunLoopRef](https://developer.apple.com/documentation/corefoundation/cfrunloop)类的指针。

### Getting a Run Loop Object
获取当前线程的runloop对象，你使用下面的一种方式：

*	在Cocoa Application中，使用[NSRunLoop](https://developer.apple.com/documentation/foundation/runloop/1412291-current)的类方法[currentRunLoop](https://developer.apple.com/documentation/foundation/runloop)，来获取`NSRunLoop`对象。
* 使用[CFRunLoopGetCurrent](https://developer.apple.com/documentation/corefoundation/1542428-cfrunloopgetcurrent)方法。

尽管它们不是无缝桥接的类，当需要的时候你可以从`NSRunLoop`对象获得一个`CFRunLoopRef`指针。`NSRunLoop`类定义了一个[getCFRunLoop](https://developer.apple.com/documentation/foundation/nsrunloop/1410140-getcfrunloop)方法，它返回了一个`CFRunLoopRef`类型，这你可以传入到Core Foundation中。因为两个对象指向同一个run loop，需要的时候你可以混合使用。

### Configuring the Run Loop
当你在自定义线程中跑run loop的时候，你至少添加一个input source或timer到run loop中，如果一个*run loop*没有任何source来监听，当你尝试运行它的时候，它会立即退出。如何向run loop中添加一个source，请看这里[Configuring Run Loop Sources](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW7)。

除了装载sources，你也可以装载run loop observers，使用它们来监听run loop的执行步骤。为了装载run loop observer，你创建了一个[CFRunLoopObserverRef](https://developer.apple.com/documentation/corefoundation/cfrunloopobserver)指针，并且使用[CFRunLoopAddObserver](https://developer.apple.com/documentation/corefoundation/1542504-cfrunloopaddobserver)方法并把它添加到你的run loop中。Run loop observer必须要使用Core Foundation来创建。

下面展示了在一个线程中，添加一个run loop observer到run loop中的主要的代码部分。

```
- (void)threadMain
{
	//The application use garbage collection,so no autorelease pool is needed.
	NSRunLoop *myRunLoop = [NSRunLoop currentRunLoop];	
	//Create a run loop observer and attach it to the run loop.
	CFRunLoopObserverContent context = {0,self,NULL,NULL,NULL};
	CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,kCFRunLoopAllActivities,YES,0,&myRunLoopObserver,&context);
	if(observer)
	{
		CFRunLoopRef cfLoop = [myRunLoop getCFRunLoop];
		CFRunLoopAddObserver(cfLoop,observer,kCFRunLoopDefaultMode);		
	}	
	//Create and schedule the timer.
		[NSTimer scheduledTimerWithTimeInterval:0.1 target:self selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
		NSInteger loopCount = 10;
	do 
	{
		//Run the run loop 10 times to let the timer fire
		[myRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1]];
		loopCount--;
	}
	while(loopCount);
}
```
当为一个活跃的线程配置run loop的时候，至少添加一个input source到接受的消息。尽管你能够只需要通过一个timer来启动run loop中，但是，一旦timer触发之后，它就会无效的，这会引起run loop的退出。传入一个repeat的timer可以保持run loop运行很长一段时间。但是将会周期性的调用触发的timer来唤醒你的线程。这是维持线程中<i>run loop</i>的另一种方式，一个input source等待事件的到达，让你的线程处于睡眠中，直到它被唤醒。

### Starting the Run Loop
一个run loop必须装载至少一个input source或者timer来。如果没有，run loop立即退出。
下面有几种办法开启一个run loop。

* 无条件的启动run loop
* 设置一个超时时间启动run loop
* 通过一种特殊的模式启动run loop

无条件的启动<i>runloop</i>是最简单的，但是这也是最不推荐的一种方式。无条件的启动你的runloop，把线程作为<i>runloop</i>的一个参数。这种方式对runloop的控制权很小。你可以添加或移除input source和timers，唯一停止<i>runloop</i>的方式就是kill。这种启动方式没有办法在自定义mode。

除了无条件的启动一个runloop，使用time out value来开始一个<i>runloop</i>更好。当你使用time out value，runloop运行直到事件到达或者超时。如果事件抵达，事件会被派发到一个handler来处理，然后run loop退出。你的code又能够重新开始runloop去处理下一个事件。如果超时，你可以简单的重启runloop，或者利用这段时间做任何需要的事。

除了time out value，你也可以使用一个指定的mode来运行runloop。Modes和timeout value并不是互斥的。当运行runloop的时候也可以同时使用它们。Modes限制了source的type。详细被描述在[Run Loop Modes](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW12)。

下面的code主要部分展示了runloop的基本结构。本质上，你添加你的input source和timer到runloop。重复调用其中一个routine来开始一个<i>run loop</i>。每次run loop routine 返回的时，你检查<i>run lop</i>是否满足某些条件导致退出thread。例子中使用Core Foundation的run loop routines，以至于它能检查返回的结果并检测run loop为什么退出。你也能使用[NSRunLoop](https://developer.apple.com/documentation/foundation/runloop)的方法以一种相似的方式来运行runloop，如果你正在使用Cocoa框架中的NSRunLoop，它是不需要检查return value的。

```
- (void)skeletonThreadMain {
	//Set up an autorelease pool here if not using garbage collection.
	BOOL done = NO;
	//Add your sources or timer to the run loop and do any other setup
	do
	{
		//Start the run loop but return after earch source is handled
		SInt32 result = CFRunLoopRunInMode(kCFRunLoopDefaultMode,10,YES)
		
		//If a source explicitly stopped the run loop,or if there no sources or timers,go head and exit.
		if ((result == kCFRunLoopRunStopped) || (result == kCFRunLoopRunFinished))
		done = YES;
		
		//Check for any other exit conditions here and set the done variable as needed
	}
	while(!done);
	//Clean up code here. Be sure to release any allocated autorelease pools.
}
```
递归运行run loop也是可能的。换句话说，你能够调用[CFRunLoopRun](https://developer.apple.com/documentation/corefoundation/1542011-cfrunlooprun)、[CFRunLoopRunInMode](https://developer.apple.com/documentation/corefoundation/1541988-cfrunloopruninmode),任何`NSRunLoop`的方法，从input source 或者 timer的handler routine开始runloop。当这样做的时候，你可以使用你想要的mode来运行内嵌的runloop。包括被outer run loop正在使用的mode。


### Exiting the Run Loop
有两种办法使runloop在处理事件之前退出。

* 为<i>Runloop</i>配置一个time out时间
* 主动告诉<i>Runloop</i>停止

使用time out时间是一种比较好的选择。在退出之前，指定一个time out时间让runloop完成所有的任务，包括向<i>runloop observer</i>发送通知。

使用[CFRunLoopStop](https://developer.apple.com/documentation/corefoundation/1541796-cfrunloopstop)方法停止一个<i>runloop</i>，产生一个类似于超时的结果。使用runloop发送完剩下的runloop通知然后退出。唯一不同的就是只有当你无条件的启动<i>Runloop</i>的时候才能使用这种方式退出。

尽管移除runloop的input source和timers可以引起runloop的退出，但是这不是一种稳定的方式。一些系统一般添加input source到run loop来处理需要的事件。因为你的code不需要关心这些input source。所以你不能移除input sources，让<i>run loop</i>退出。

### Thread Safety and Run Loop Objects
线程安全取决于你使用操作<i>runloop</i>的API。在Core Foundation中的方法一般是线程安全的，可以被任何其他线程调用。如果你正在执行操作，这些操作改变了runloop的配置。无论什么时候，这样做任然是一个很好的实践。

Cocoa的[NSRunLoop](https://developer.apple.com/documentation/foundation/runloop)类。Cocoa的NSRunLoop和Core Foundation不一样，不是线程安全的。如果你使用`NSRunLoop`类来修改你的runloop，你应该在原来的线程中做这些操作。在其他线程添加一个input source或者timer到原来runloop可能引起crash，或者一个意想不到的行为。