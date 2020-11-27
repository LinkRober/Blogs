+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "zip"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


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
同`zip`压缩的信号只有等所有的自信号都`sendNext`之后才会执行订阅回调。

问题：
1. 如何确保每个信号都发送完，才执行后压缩信号的订阅回调

<!--more-->

```
答：
+ (instancetype)join:(id<NSFastEnumeration>)streams block:(RACStream * (^)(id, id))block {
    RACStream *current = nil;
	// Creates streams of successively larger tuples by combining the input
	// streams one-by-one.
	for (RACStream *stream in streams) {
		// For the first stream, just wrap its values in a RACTuple. That way,
		// if only one stream is given, the result is still a stream of tuples.
		if (current == nil) {
			current = [stream map:^(id x) {
				return RACTuplePack(x);
			}];

			continue;
		}

		current = block(current, stream);
	}
	    ....
	    ....
	    ....
}
```
第一次`map`操作信号1会生成一个新信号new1，new1和信号2经过`block(current,stream)`又生成一个新信号new2，以此类推。
这两种操作都会对原始信号进行订阅，如果接受不到请求就不会继续下去。

2. 信号是如何被压平的

在发生zs的订阅之后，通过`RACTuplePack(x)`，生成一个元组，然后不断循环，形成元组套值(元组+值)的形式
```
<RACTuple: 0x60000236c720> (
    <RACTuple: 0x600002375d00> (1),
    2
)
```

最后通过递归算法将值从元组套元组的组合里一一提取出来，这样信号就压平了。
```
NSMutableArray *values = [[NSMutableArray alloc] init];
while (xs != nil) {
    [values insertObject:xs.last ?: RACTupleNil.tupleNil atIndex:0];
	xs = (xs.count > 1 ? xs.first : nil);
}
return [RACTuple tupleWithObjectsFromArray:values];
```

