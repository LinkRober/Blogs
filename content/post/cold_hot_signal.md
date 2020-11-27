+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "冷信号和热信号构建触发流程"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


提到RAC就离不开冷热信号。在实际的开发中如果不理清两者的用法就会出问题。

冷信号像专车，从你打车（订阅）开始，从出发地点（sendNext）发往你在的地方。它是一对一的，不会错过。

热信号像公交车，在你等车（订阅）之前可能已经发车了（sendNext）；不仅你可以上，其他人也能上，它是一对多的；错过了这班车（假设只有一班）就没了。

<!--more-->

```
//冷信号
    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@1];
        return nil;
    }];

    [[RACScheduler mainThreadScheduler] afterDelay:1.1 schedule:^{
        [signal subscribeNext:^(id x) {
            NSLog(@"cold signal  recveive: %@", x);
        }];
    }];
```
>输出: cold signal  recveive: 1

```
//热信号
        RACMulticastConnection *connection =  [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
            [subscriber sendNext:@1];
            return nil;
        }] publish];
        [connection connect];

        RACSignal *signal = connection.signal;

        [[RACScheduler mainThreadScheduler] afterDelay:1.1 schedule:^{
            [signal subscribeNext:^(id x) {
                NSLog(@"hot signal recveive: %@", x);
            }];
        }];
```
>没有输出，公交车比你早到了1.1s

冷信号转换成热信号经过了两步`[RACSignal publish]`和`[RACMulticastConnection connect]`，
第一步将信号`signal`和`subject`进行绑定，返回一个`RACMulticastConnection`对象。

```
- (RACMulticastConnection *)multicast:(RACSubject *)subject {
	[subject setNameWithFormat:@"[%@] -multicast: %@", self.name, subject.name];
	RACMulticastConnection *connection = [[RACMulticastConnection alloc] initWithSourceSignal:self subject:subject];
	return connection;
}
```
第二步立即触发信号的订阅，公交车开始发车了
```
- (RACDisposable *)connect {
	BOOL shouldConnect = OSAtomicCompareAndSwap32Barrier(0, 1, &_hasConnected);

	if (shouldConnect) {
		self.serialDisposable.disposable = [self.sourceSignal subscribe:_signal];
	}

	return self.serialDisposable;
}
```
那么`RACMulticastConnection`是做什么的呢?
它将信号（Signal）和订阅（subject）包装，为一个或多个订阅者提供服务（一对多）。当信号存在潜在的副作用或者接受事件不超过1次（eg.`[subscriber sendNext:@1]`）的时候使用它。

![](https://note.youdao.com/yws/api/personal/file/WEBb47d084e7444069fc09c784160fa22be?method=download&shareKey=44e2ac121991c7bfe127a3f1b3539762)
