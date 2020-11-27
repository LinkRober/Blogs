+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "CombineLatest、Zip和Merge的区别"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


#### 信号触发方式

##### merge

只要`merge`之后生成的信号被订阅就会**自动**触发所有压缩信号的订阅回调，如果靠前的信号出现了`error`后面的信号不再发送。


核心方法：`- (instancetype)flatten`</br>
值：多次收到，分开的
```
 1
 2
```

```
RACSignal *s1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
  [subscriber sendNext:@"1"];
    return nil;
}];

RACSignal *s2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"2"];
    return nil;
}];

[[RACSignal merge:@[s1,s2]] subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];

```

<!--more-->

##### combineLatest

必须等到**所有**的信号都成功发送才能触发`combineLatest`生成的新信号的订阅回调。如果中间出现有信号error或者complete，新信号将收不到回调。

核心方法：
`+ (instancetype)join:(id<NSFastEnumeration>)streams block:(RACStream * (^)(id, id))block`

`- (RACSignal *)combineLatestWith:(RACSignal *)signal`
</br>

值：一次收到，通过`RACTuple`聚合的
```
<RACTuple: 0x60000318c780> (
    1,
    2
)
```

```
RACSubject *s1 = [RACSubject subject];
[s1 subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];

RACSubject *s2 = [RACSubject subject];
[s2 subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];

[[RACSignal combineLatest:@[s1,s2]] subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];

[s1 sendNext:@"1"];
[s2 sendNext:@"2"];

```

##### zip

必须等到**所有**的信号都成功发送才能触发`zip`生成的新信号的订阅回调(和`combineLatest`类似)。如果中间出现有信号error或者complete，新信号将收不到回，和`combineLatest`不同的是，同一时间返回多个信号的值只取第一个，而`combineLatest`是取最新的值

```
//zip
....
void (^sendNext)(void) = ^{
			@synchronized (selfValues) {
				if (selfValues.count == 0) return;
				if (otherValues.count == 0) return;
				//去数组第一个值
				RACTuple *tuple = [RACTuple tupleWithObjects:selfValues[0], otherValues[0], nil];
				[selfValues removeObjectAtIndex:0];
				[otherValues removeObjectAtIndex:0];

				[subscriber sendNext:tuple];
				sendCompletedIfNecessary();
			}
		};
....
```

```
//combineLatestWith
...
void (^sendNext)(void) = ^{
			@synchronized (disposable) {
				if (lastSelfValue == nil || lastOtherValue == nil) return;
				[subscriber sendNext:[RACTuple tupleWithObjects:lastSelfValue, lastOtherValue, nil]];
			}
		};
...
```

核心方法：
1. `+ (instancetype)join:(id<NSFastEnumeration>)streams block:(RACStream * (^)(id, id))block`
2. `- (RACSignal *)zipWith:(RACSignal *)signal`

值：一次收到，通过`RACTuple`聚合的
```
 <RACTuple: 0x600002668e20> (
    1,
    2,
    3
)
```
```
RACSignal *s1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
   [subscriber sendNext:@"1"];
    return nil;
}];
RACSignal *s2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"2"];
    return nil;
}];
RACSignal *s3 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"3"];
    return nil;
}];

RACSignal *zs = [RACSignal zip:@[s1,s2,s3]];

[zs subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];
```


