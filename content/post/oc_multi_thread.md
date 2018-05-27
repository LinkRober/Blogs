+++
keywords = ["NSOperation","GCD","NSThread"]
tags = ["多线程","objectc"]
categories = ["ios"]
date = "2018-05-27T00:00:00Z"
title = "Object-C多线程总结"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true

+++

### 目录

- 多线程概念
- NSOperation
- GCD(Grand Central Dispatch)
- NSThread
- 资源竞争

<!--more-->


### 多线程概念

线程也叫轻量级进程，是程序执行流的最小单元。
线程组成：**线程ID**、**指令指针**、**寄存器集合**、**堆栈**。
线程是进程中的一个实体，是被系统独立调度和分派的基本单位。

线程自己不拥有系统资源，只拥有一点在运行中必不可少的资源，但它可与同属于一个进程的其他线程共享进程所拥有的全部资源。一个线程可以创建和撤销另一个线程，同一进程中的多个线程之间可以并发执行。

线程有就绪、阻塞和运行三种基本状态。<br>
**就绪**：是指线程具备运行的所有条件，逻辑上可运行，在等待处理机。<br>
**运行**：线程占有处理机正在运行<br>
**阻塞**：线程在等一个事件（如信号量），逻辑上不可执行

每一个程序都至少有一个线程，若程序只有一个线程，那就是程序本身。在单个程序中同时运行多个线程完成不同的工作，称为**多线程**。

<br>
### NSOperation
因为`NSOperation`是一个抽象类，我们不能直接使用它，我们一般使用系统定义好了的两个子类`NSInvocationOperation`和`NSBlockOperation`。尽管`NSOperation`很抽象，但是它的实现很优雅，能让你安全的执行任务。

它是一次性对象（single-shot object），一个任务执行结束了，就不能再次执行。我们通常将它添加到`NSOperationQueue`来执行任务。`NSOperationQueue`将任务放到其他线程或者直接调动`GCD`来执行任务。如果你不想使用`NSOperationQueue`可以直接通过`NSOperation`的`start`方法来执行。但是这样会给你的code带来更多的负担。如果执行的时候线程不在一个就绪状态就会引起异常(`NSOperation`存在依赖)。

<br>

#### NSBlockOperation&NSInvocationOperation
添加任务
```
//selector
- (nullable instancetype)initWithTarget:(id)target selector:(SEL)sel object:(nullable id)arg;
//block
+ (instancetype)blockOperationWithBlock:(void (^)(void))block;
```
<br>

操作
```
//开始执行任务
- (void)start
//可以取消线程操作对应方法
- (void)cancel
//添加依赖
- (void)addDependency:(NSOperation *)op
//移除依赖
- (void)removeDependency:(NSOperation *)op
```

<br>
#### NSOperationQueue
`NSOperation`的队列，通过设置并发数量来控制异步还是同步，属性为`maxConcurrentOperationCount`

添加任务
```
- (void)addOperation:(NSOperation *)op;
```

```
- (void)addOperations:(NSArray<NSOperation *> *)ops waitUntilFinished:(BOOL)wait API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));
```

```
- (void)addOperationWithBlock:(void (^)(void))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));

```
>添加完之后会立即执行

取消任务
```
- (void)cancelAllOperations;
```

等待

```
- (void)waitUntilAllOperationsAreFinished;
```
>会使当前线程（例如，主线程）忙等，直到所有任务完成

<br>
### GCD (Grand Central Dispatch)
GCD将语言特性、OC运行时和系统提供的优化结合在一起，充分发挥系统的多核特性。BSD子系统，Core Foundation和Cocoa API都是用了CGD来帮助iOS系统和App跑的更快、更高效、响应更及时。GCD在系统层面更好的协调了所有在run的App，使它们在系统资源的利用上达到了一个平衡。<br>

当你使用Object-C的编译器编译你的App，所有的GCD对象都是Object-C对象。在RAC环境下，引用和释放也是自动管理的。在非ARC环境下需要手动管理。（dispatch_retain和 dispatch_release）

`CGD`每个任务都要放到一个队列中去执行，队列分两种串行队列和并行队列。任务也分两种同步和异步。



  | 串行队列 | 并行队列
-------|--------|----
同步线程 | 会阻塞当前线程；不会开辟新的线程；任务在当前的线程串行执行  | 会阻塞当前线程；不会开辟新的线程；任务在当前的线程同步执行
异步线程 | 不会阻塞当前线程；会开辟新的线程；任务在新的线程串行执行  | 不会阻塞当前线程；会开辟新的线程；任务在新的线程异步执行

<br>
#### 死锁
```
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"sync - %@", [NSThread currentThread]);
});
```
因为是同步任务，主线程停下，等待队列里的任务完成，因为队列是主队列，跑的是主线程，这时候主线程处于等待状态，就出现了互相等待的情况。

<br>
#### dispatch_group
`dispatch_group`能够在一个`Queue`的所有任务结束之后得到通知，执行相关代码

```
    dispatch_group_t anchorGroup = dispatch_group_create();

```

```
dispatch_group_notify(anchorGroup, anchorQueue, ^{
        //刷新第一个cell
        @strongify(self);
        [self.tableView reloadRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:0 inSection:0]] withRowAnimation:UITableViewRowAnimationFade];
    });
```
<br>
#### dispatch_group_enter&&dispatch_group_leave

当GCD的任务是异步的时候，如果想要在所有任务结束之后得到通知，就要在各个任务的前后加上`dispatch_group_enter`和`dispatch_group_leave`。他们需要传入`group`作为参数

```
dispatch_group_enter(anchorGroup);
    dispatch_async(anchorQueue, ^{
        [[XYLeaderboardService share] anchorCharmLeaderboard:XYLeaderboardCharmMonthType succeed:^(NSArray<XYLeaderboardItem *> *items) {
            @strongify(self);
            [self.datasource firstObject].secondLeaderboard = items;
            dispatch_group_leave(anchorGroup);
        } failure:^(XYNetError *error) {
            //
            dispatch_group_leave(anchorGroup);
        }];
    });
```
<br>
#### dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t block);
等待queue中的任务全部结束，异步执行block中的任务

<br>
#### dispatch_barrier_sync(dispatch_queue_t queue,DISPATCH_NOESCAPE dispatch_block_t block);
等待queue中的任务全部结束，同步执行block中的任务

### NSThread
在执行一个长任务的时候比较有优势，其他的和`NSOperation`比较像。

#### 创建NSThread

```
NSThread *thread = [[NSThread alloc] initWithBlock:^{

    }];
```
```
NSThread *thread = [[NSThread alloc] initWithTarget:<#(nonnull id)#> selector:<#(nonnull SEL)#> object:<#(nullable id)#>];

```

#### 运行NSThread
```
NSThread *thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"thread测试%@",[NSThread currentThread]);
    }];
    [thread start];

//直接运行
[NSThread detachNewThreadWithBlock:^{
        NSLog(@"thrad直接执行%@",[NSThread currentThread]);
    }];
```

#### 停止NSThread

```
//退出当前线程
[NSThread exit];
//取消该线程
NSThread *thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"thread测试%@",[NSThread currentThread]);
    }];
[thread cancel];

//阻塞当前线程直到某个时间
[NSThread sleepUntilDate:<#(nonnull NSDate *)#>];

//使当前线程沉睡多少时间
NSLog(@"===============start sleep================");
[NSThread sleepForTimeInterval:5];
NSLog(@"==============end sleep=================");

```

#### 检测当前NSThread的状态
```
//是否正在执行
[thread isExecuting];
//是否完成
[thread isFinished];
//是否被取消
[thread isCancelled];
```

#### Priority
```
//设置当前线程的优先级
[NSThread setThreadPriority:<#(double)#>];
//查看当前线程的优先级
[NSThread threadPriority];
```

### 资源竞争
[iOS多线程中的锁](https://linkrober.github.io/bookshelf/2017/12/ios%E5%A4%9A%E7%BA%BF%E7%A8%8B%E4%B8%AD%E7%9A%84%E9%94%81/)
