+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "Connect、AutoConnect、RACReplaySubject、Replay"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++

#### Connect
将冷信号包装成热信号，初始化冷信号，调用`publish`方法会自动生成一个`RACMulticastConnection`，该对象持有了原始信号和一个热信号。


```
- (RACMulticastConnection *)publish {
	RACSubject *subject = [[RACSubject subject] setNameWithFormat:@"[%@] -publish", self.name];
	RACMulticastConnection *connection = [self multicast:subject];
	return connection;
}
- (RACMulticastConnection *)multicast:(RACSubject *)subject {
	[subject setNameWithFormat:@"[%@] -multicast: %@", self.name, subject.name];
	RACMulticastConnection *connection = [[RACMulticastConnection alloc] initWithSourceSignal:self subject:subject];
	return connection;
}
```
调用`connect`方法来触发订阅，注意调用一次触发一次。

<!--more-->

```
- (RACDisposable *)connect {
	BOOL shouldConnect = OSAtomicCompareAndSwap32Barrier(0, 1, &_hasConnected);

	if (shouldConnect) {
		self.serialDisposable.disposable = [self.sourceSignal subscribe:_signal];
	}

	return self.serialDisposable;
}

```
![](https://note.youdao.com/yws/api/personal/file/WEB322c067dfd25b07390b5016979f95684?method=download&shareKey=18be9a0156c074ce3ed310a30cf3d782)


#### Autoconnect
在`publish`之后，`RACMulticastConnection`调用`autoconnect`，生成一个信号，只有当生成的信号订阅之后才会触发其他订阅者开始接受订阅消息。
```
- (RACSignal *)autoconnect {
	__block volatile int32_t subscriberCount = 0;

	return [[RACSignal
		createSignal:^(id<RACSubscriber> subscriber) {
			OSAtomicIncrement32Barrier(&subscriberCount);

			RACDisposable *subscriptionDisposable = [self.signal subscribe:subscriber];
			RACDisposable *connectionDisposable = [self connect];

			return [RACDisposable disposableWithBlock:^{
				[subscriptionDisposable dispose];

				if (OSAtomicDecrement32Barrier(&subscriberCount) == 0) {
					[connectionDisposable dispose];
				}
			}];
		}]
		setNameWithFormat:@"[%@] -autoconnect", self.signal.name];
}
```
![](https://note.youdao.com/yws/api/personal/file/WEBfd22a762ef7800a5d21b2fd823b26905?method=download&shareKey=0d72e57669a7945987449fb7dd3179f3)


#### RACMulticastConnection & RACReplaySubject
上面在`connect`之后的订阅者都不到订阅值，使用`RACReplaySubject`可以让后面的订阅者也能收到订阅值。

例子：
```
RACSignal *sourceSignal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
    [subscriber sendNext:@{@"id":@"1"}];
    return nil;
}];

RACMulticastConnection *connection = [sourceSignal multicast:[RACReplaySubject subject]];

[connection.signal subscribeNext:^(id  _Nullable x) {
    NSLog(@"product: %@", x);
}];

[connection connect];

[connection.signal subscribeNext:^(id  _Nullable x) {
    NSNumber *productId = [x objectForKey:@"id"];
    NSLog(@"productId: %@", productId);
}];

```
![](https://note.youdao.com/yws/api/personal/file/WEBeaf769304dcdeb50aedaf81ef877ce54?method=download&shareKey=825769b6f66cab9528c1aa2337763104)

#### Replay
快速生成一个基于`RACReplaySubject`创建的热信号。
```
- (RACSignal *)replay {
	RACReplaySubject *subject = [[RACReplaySubject subject] setNameWithFormat:@"[%@] -replay", self.name];

	RACMulticastConnection *connection = [self multicast:subject];
	[connection connect];

	return connection.signal;
}
```
例子：
```
RACSignal *sourceSignal = [[RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        [subscriber sendNext:@{@"id":@"1"}];
        return nil;
    }] replay];

    [sourceSignal subscribeNext:^(id  _Nullable x) {
        NSLog(@"product: %@", x);
    }];

    [sourceSignal subscribeNext:^(id  _Nullable x) {
        NSNumber *productId = [x objectForKey:@"id"];
        NSLog(@"productId: %@", productId);
    }];
```
![](https://note.youdao.com/yws/api/personal/file/WEBdddd77c6ae072d234b09b7b5e6e3e17c?method=download&shareKey=35c80261e0e41092892c011b92cbb488)