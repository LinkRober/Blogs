+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "信号中的信号-flatten"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


```
    RACSignal *targetSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@"1"];
        return nil;
    }];
    
    RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    //发送一个信号
        [subscriber sendNext:targetSignal];
        return nil;
    }];
    
    RACSignal *flattenSignal = [signal flatten];
    [flattenSignal subscribeNext:^(id x) {
        NSLog(@"%@",x);//1
    }];

```

<!--more-->

`[signal flatten]`做了什么，可以看出最终返回的是一个信号，深入方法里面，进行了如下的方法调用

1. `- (instancetype)flatten`
2. `- (instancetype)flattenMap:(RACStream * (^)(id value))block`
3. `- (RACSignal *)bind:(RACStreamBindBlock (^)(void))block`

最终：
```
return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {....}
```

在这个方法里面，首先对自己就行了订阅，于是执行了`[subscriber sendNext:targetSignal];`然后两层回调，`bindingBlock(x,&stop)`和`block(value)`。返回信号`targetSignal`。将信号作为`void (^addSignal)(RACSignal *) = ^(RACSignal *signal){}`的入参，其实是在这里面执行`targetSignal`订阅，得到了值**1**，最后通过原来的`[subscriber sendNext:x];`将值返回。

这时候你可能想，这里只嵌套了一层信号，如果嵌套两层怎么处理。你可以类比成递归，将`RACSignal *flattenSignal = [signal flatten]`作为递归中的一步，然后对层层信号进行`flatten`操作，最后就能得到你想要的结果。

```
//三个信号
    RACSignal *signal1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@"1"];
        return nil;
    }];
    
    RACSignal *signal2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:signal1];
        return nil;
    }];
    
    RACSignal *faltSignal2 = [signal2 flattenMap:^RACStream *(id value) {
        NSLog(@"");
        return value;
    }];
    
    
    RACSignal *param3 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:faltSignal2];
        return nil;
    }];
    
    RACSignal *flatSignal3 = [param3 flattenMap:^RACStream *(id value) {
        NSLog(@"");
        return value;
    }];
    
    [flatSignal3 subscribeNext:^(id x) {
        NSLog(@"");
    }];
```
最后输出了1