+++
tags = ["ReactiveCocoa"]
categories = ["ios"]
date = "2020-11-27T00:00:00Z"
title = "concat、catchTo、never、takeUntil"
draft = false
comments = true
showPagination = true
showTags = true
showSocial = true
showDate = true
+++


1. `- (RACSignal *)concat:(RACSignal *)signal`

```
//源码
- (RACSignal *)concat:(RACSignal *)signal {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *serialDisposable = [[RACSerialDisposable alloc] init];

		RACDisposable *sourceDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			RACDisposable *concattedDisposable = [signal subscribe:subscriber];
			serialDisposable.disposable = concattedDisposable;
		}];

		serialDisposable.disposable = sourceDisposable;
		return serialDisposable;
	}] setNameWithFormat:@"[%@] -concat: %@", self.name, signal];
}
```

<!--more-->

多个信号顺序输出，只有当前一个信号`sendCompleted`后一个信号才会执行。
```
//eg.
RACSignal *s1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"1"];
    [subscriber sendCompleted];
    return nil;
}];

RACSignal *s2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"2"];
    return nil;
}];

[[s1 concat:s2] subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];
```
输出：

2020-07-06 17:39:06.542592+0800 C-41[3323:243230] 1 </br>
2020-07-06 17:39:06.542817+0800 C-41[3323:243230] 2

2. `- (RACSignal *)catchTo:(RACSignal *)signal`
```
//源码
- (RACSignal *)catch:(RACSignal * (^)(NSError *error))catchBlock {
	NSCParameterAssert(catchBlock != NULL);

	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *catchDisposable = [[RACSerialDisposable alloc] init];

		RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			RACSignal *signal = catchBlock(error);
			NSCAssert(signal != nil, @"Expected non-nil signal from catch block on %@", self);
			catchDisposable.disposable = [signal subscribe:subscriber];
		} completed:^{
			[subscriber sendCompleted];
		}];

		return [RACDisposable disposableWithBlock:^{
			[catchDisposable dispose];
			[subscriptionDisposable dispose];
		}];
	}] setNameWithFormat:@"[%@] -catch:", self.name];
}
```
当某个信号发生错误时`senderError`，将替补信号的值作为备选值。

```
//eg.
RACSignal *s1 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendError:nil];
    return nil;
}];

RACSignal *s2 = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"替补队员跟上"];
    return nil;
}];

[[s1 catchTo:s2] subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];
```
输出：

2020-07-06 17:48:06.347187+0800 C-41[3418:248921] 替补队员跟上


3. `+ (RACSignal *)never`
```
//源码
+ (RACSignal *)never {
	return [[self createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
		return nil;
	}] setNameWithFormat:@"+never"];
}
```
订阅者永远收不到订阅值；没有错误；没有完成。



4. `- (RACSignal *)takeUntil:(RACSignal *)signalTrigger`

```
//源码
- (RACSignal *)takeUntil:(RACSignal *)signalTrigger {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
		void (^triggerCompletion)(void) = ^{
			[disposable dispose];
			[subscriber sendCompleted];
		};

		RACDisposable *triggerDisposable = [signalTrigger subscribeNext:^(id _) {
			triggerCompletion();
		} completed:^{
			triggerCompletion();
		}];

		[disposable addDisposable:triggerDisposable];

		RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			[disposable dispose];
			[subscriber sendCompleted];
		}];

		[disposable addDisposable:selfDisposable];

		return disposable;
	}] setNameWithFormat:@"[%@] -takeUntil: %@", self.name, signalTrigger];
}
```
正在进行的信号叫做A信号，遇到信号B被订阅就停止。
首先，通过A信号创建了一个中间信号A1。开始时对A信号进行了了订阅，并将值作为A1信号的输出。当B信号被订阅时，会触发A1信号的`sendCompleted`操作。这样无理A怎么发消息，A1都不会转发。

```
//eg.
RACSubject *untilSignal = [RACSubject subject];
RACSubject *afterUntilSignal = [RACSubject subject];

[[afterUntilSignal takeUntil:untilSignal] subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];

[untilSignal subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];

[afterUntilSignal sendNext:@"1"];
[afterUntilSignal sendNext:@"2"];
[untilSignal sendNext:@"stop"];
[afterUntilSignal sendNext:@"3"];
```
输出

2020-07-06 17:42:51.878488+0800 C-41[3354:245702] 1</br>
2020-07-06 17:42:51.878642+0800 C-41[3354:245702] 2</br>
2020-07-06 17:42:51.878854+0800 C-41[3354:245702] stop