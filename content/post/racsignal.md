+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "RACSignal"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


#### 创建信号

```

RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@"1"];
        return nil;
    }];

```

分析

```
//RACSignal.m
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	return [RACDynamicSignal createSignal:didSubscribe];
}
```

<!--more-->

通过子类创建信号

```
//RACDynamicSignal.m
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
	signal->_didSubscribe = [didSubscribe copy];
	return [signal setNameWithFormat:@"+createSignal:"];
}
```
创建一个RACDynamicSignal，将回调`didSubscribe`保存在`_didSubscribe`中。

#### 订阅信号
```
[signal subscribeNext:^(id x) {
        
}];
```
分析
```
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
	NSCParameterAssert(nextBlock != NULL);
	
	RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
	return [self subscribe:o];
}

```
创建一个订阅者`RACSubscriber`，将回调`nextBlock`作为订阅者的初始入参，最后订阅RACSubscriber。

```
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);
	//创建销毁对象，负责销毁信号
	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	//将RACSubscriber类型的订阅者转换成RACPassthroughSubscriber，为什么呢
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];
	
	//创建自旋锁，防止资源竞争，懒加载订阅者集合(_subscribers),
	OSSpinLockLock(&_subscribersLock);
	if (_subscribers == nil) {
		_subscribers = [NSMutableArray arrayWithObject:subscriber];
	} else {
		[_subscribers addObject:subscriber];
	}
	OSSpinLockUnlock(&_subscribersLock);
	
	@weakify(self);
	//创建销毁对象（和上面的有什么不同？？？，看下面），当信号被销毁，从订阅者集合里面移除subscriber
	RACDisposable *defaultDisposable = [RACDisposable disposableWithBlock:^{
		@strongify(self);
		if (self == nil) return;

		BOOL stillHasSubscribers = YES;

		OSSpinLockLock(&_subscribersLock);
		{
			// Since newer subscribers are generally shorter-lived, search
			// starting from the end of the list.
			NSUInteger index = [_subscribers indexOfObjectWithOptions:NSEnumerationReverse passingTest:^ BOOL (id<RACSubscriber> obj, NSUInteger index, BOOL *stop) {
				return obj == subscriber;
			}];

			if (index != NSNotFound) {
				[_subscribers removeObjectAtIndex:index];
				stillHasSubscribers = _subscribers.count > 0;
			}
		}
		OSSpinLockUnlock(&_subscribersLock);
		
		if (!stillHasSubscribers) {
			[self invalidateGlobalRefIfNoNewSubscribersShowUp];
		}
	}];
	
	//将创建的销毁对象添加到一个管理类中
	[disposable addDisposable:defaultDisposable];

    //通过`didSubscribe`拿到订阅者对应的innerDisposable，添加到disposable中，
    //最后将schedulingDisposabl也添加到disposable中
	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}
	
	return disposable;
}
```
信号被订阅之后的销毁流程。首先创建了一个销毁者管理类`disposable`，以subscriber为入参生成一个新的subscriber（RACPassthroughSubscriber），老的订阅者和销毁者管理类都在这里面管理。
然后将`defaultDisposable`、`innerDisposable`、 `schedulingDisposable`都放到管理类中进行管理



#### 问题
1. 如何做到一订阅，就开始执行`[subscriber sendNext:xxx];`

本质是调用了block。在创建的时候，block会被属性引用
```
signal->_didSubscribe = [didSubscribe copy];
```
在订阅方法的最后，调用代码块`didSubscribe`
```
	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}

```
