+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "SwitchToLatest"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


`- (RACSignal *)switchToLatest`和`- (RACSignal *)flatten`的功能类似，都可以将信号中的信号的值取出来。不同的是，前者如果订阅者处理多个信号，只有最后一个信号的值能收到，之前的信号会被销毁。而后者则都能收到所有值。

<!--more-->

代码
```
RACSignal *signal1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
   
   
    [subscriber sendNext:@"nancy"];
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        
         [subscriber sendNext:@"botwen"];
    });
    return nil;
}];
    
RACSignal *signal2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"jiang"];
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
         [subscriber sendNext:@"longjian"];
    });
    return nil;
}];
    
RACSignal *signal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
    [subscriber sendNext: signal1];
    [subscriber sendNext: signal2];
    [subscriber sendCompleted];
    
    return nil;
}];

RACSignal *newSignal = [signal switchToLatest];
[newSignal subscribeNext:^(id  _Nullable x) {
    NSLog(@"-=-=======-=%@", x);
}];
```

输出：
```
-=-=======-=nancy
-=-=======-=jiang
-=-=======-=longjian
```
`botwen`没有输出，因为`signal1`已经被销毁。
这里有一个难点，**为什么`nancy`输出了**。要知道原因，我们必须要清楚的理解整个信号的创建和销毁时机。

原信号O通过`- (RACSignal *)switchToLatest`方法生成了一个新信号A，当A被订阅时，O信号通过
```
RACMulticastConnection *connection = [self publish];
```
生成了一个热信号，然后调用
```
- (__kindof RACStream *)flattenMap:(__kindof RACStream * (^)(id value))block
```
来实现信号中信号的压平，这里和`- (__kindof RACStream *)flatten`一样，最大的不同是`block`的实现，在`- (RACSignal *)switchToLatest`中，`block`里面调用了`takeUntil`方法。
```
RACSignal *s = [connection.signal
flattenMap:^(RACSignal *x) {
    NSCAssert(x == nil || [x isKindOfClass:RACSignal.class], @"-switchToLatest requires that the source signal (%@) send signals. Instead we got: %@", self, x);
    return [x takeUntil:[connection.signal concat:[RACSignal never]]];
}];
```
当信号`[connection.signal concat:[RACSignal never]]`被**订阅**的时候，通过`takeUntil`生成的信号B会被销毁。这也是上面销毁`signal1`的地方。

那么在什么时机触发销毁动作呢？这里涉及到热信号。每个热信号都有一个订阅者的数组，当热信号受到订阅者发送的消息时，会遍历这个数组，一一调用。
```
- (void)enumerateSubscribersUsingBlock:(void (^)(id<RACSubscriber> subscriber))block {
	NSArray *subscribers;
	@synchronized (self.subscribers) {
		subscribers = [self.subscribers copy];
	}

	for (id<RACSubscriber> subscriber in subscribers) {
		block(subscriber);
	}
}

- (void)sendNext:(id)value {
	[self enumerateSubscribersUsingBlock:^(id<RACSubscriber> subscriber) {
		[subscriber sendNext:value];
	}];
}
```
实例代码里热信号的订阅者数组里面最多的时候一个有3个订阅者，下面列出添加订阅者数组的地方：
```
//[self subscribeNext:^(id x)...调用1次
RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {
    // Manually check disposal to handle synchronous errors.
    if (compoundDisposable.disposed) return;

    BOOL stop = NO;
    id signal = bindingBlock(x, &stop);

    @autoreleasepool {
        if (signal != nil) addSignal(signal);
        if (signal == nil || stop) {
            [selfDisposable dispose];
            completeSignal(selfDisposable);
        }
    }
} error:^(NSError *error) {
    [compoundDisposable dispose];
    [subscriber sendError:error];
} completed:^{
    @autoreleasepool {
        completeSignal(selfDisposable);
    }
}];
```

```
//这里的[self subscribeNext:^(id x)...调用了2次，对应两次订阅者发送信号
- (RACSignal *)concat:(RACSignal *)signal {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACCompoundDisposable *compoundDisposable = [[RACCompoundDisposable alloc] init];

		RACDisposable *sourceDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			RACDisposable *concattedDisposable = [signal subscribe:subscriber];
			[compoundDisposable addDisposable:concattedDisposable];
		}];

		[compoundDisposable addDisposable:sourceDisposable];
		return compoundDisposable;
	}] setNameWithFormat:@"[%@] -concat: %@", self.name, signal];
}
```
第一次为了压平信号调
```
subscribers = [subscriber1]
```
第二次在`[subscriber sendNext: signal1];`后调用。
```
subscribers = [subscriber1,subscriber2]
```

第三次在`[subscriber sendNext: signal2];`后调用。
```
subscribers = [subscriber1,subscriber2,subscriber3]
```
在第三次调用`sendNext`的时候会遍历订阅者数组（这时候数组里面还只有两个订阅者）发送信号从而走到
```
...
RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
		void (^triggerCompletion)(void) = ^{
			[disposable dispose];
			[subscriber sendCompleted];
		};

		RACDisposable *triggerDisposable = [signalTrigger subscribeNext:^(id _) {
		    //走到这里
			triggerCompletion();
		} completed:^{
			triggerCompletion();
		}];
...

```
将`signal1`的订阅者销毁，以此类推。这也是为什么第一次两个信号的值都能收到的原因。

到这里就是`SwitchToLatest`的功能实现。

难点；

1. 理解`- (RACSignal *)takeUntil:(RACSignal *)signalTrigger`
2. 冷信号转换成热信号的实现