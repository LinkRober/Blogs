+++
date = "2017-04-03T03:05:31+08:00"
title = "消息转发"
draft = false
tags = ["runtime","oc"]
categories = [
  "Development",
]
# thumbnailImagePosition: left
# thumbnailImage: //d1u9biwaxjngwg.cloudfront.net/cover-image-showcase/city-750.jpg
+++

>描述：如果类不能执行这个方法，会执行动态**消息转发**，如果该类还是不能动态的添加方法，则走**完整的消息转发**。分两步，第一步**看看有没有其他类可以执行该方法**，如果没有走第二步，**将所有的细节封装到NSInvocation中，给接受者最后一次机会**。

演示：
动态消息转发
>在一个类`MessageObj`中定义两个方法，`testDynamicMethodForward`一个有实现方法，`start`一个没有实现的方法，调用没有实现的方法，在动态消息转发的时候将这个方法hook到已经实现的方法上：
<!--more-->
```
@interface MessageObj()
@end


@implementation MessageObj

//-(void)start{}
void testDynamicMethodForward(){
    printf("testDynamicMethodForward \n");
}

+(BOOL)resolveInstanceMethod:(SEL)sel {
    class_addMethod([self class], sel, (IMP)testDynamicMethodForward, "v@@:");
    return YES;
}
@end
```
打印如下：testDynamicMethodForward 

完整的消息转发第一步
>定义两个类，第一个类`MessageObj`有一个未实现的方法`start`，并且没有实现动态消息转发。第二个类`OtherClass`，有一个和第一个类中未实现的方法同名的方法`start`，在进行完整消息转发的第一步时，将`MessageOb`j中未实现的方法hook到，`OtherClass`的同名方法`start`中

OtherClass类
```
@interface OtherClass : NSObject

@end

@implementation OtherClass

-(void)start{
    NSLog(@"do some thing %@",[self class]);
}

@end
```
MessageObj类
```
@implementation MessageObj

//-(void)start{}

-(id)forwardingTargetForSelector:(SEL)aSelector {
    printf("%p \n",&aSelector);
    OtherClass *obj = [OtherClass new];
    return obj;
}
```
调用`start`方法
```
MessageObj *obj = [MessageObj new];
[obj start];
``` 
打印：do some thing OtherClass

完整消息转发的第二步
如果以上两步都失败了，就走到这里。定义两个类`MessageObj`、`OtherClass`，`MessageObj`中存在`OtherClass`的实例。当走到消息转发的第三步时，先进行方法重签名`-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`，再走最后的消息转发`-(void)forwardInvocation:(NSInvocation *)anInvocation`。

OtherClass类
```
@interface OtherClass : NSObject

@end
@implementation OtherClass

-(void)start{
    NSLog(@"message transform third part %@",[self class]);
}

@end
```
MessageObj类
```
@implementation MessageObj


-(instancetype)init {
    if (self = [super init]) {
        otherClass = [OtherClass new];
    }
    return  self;
}


//-(void)start{}
//方法重签名
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (!signature) {
        signature = [otherClass methodSignatureForSelector:aSelector];
    }
    return signature;
}
//最后的消息转发
-(void)forwardInvocation:(NSInvocation *)anInvocation {
    if (!otherClass) {
        [self doesNotRecognizeSelector:[anInvocation selector]];
    }
    [anInvocation invokeWithTarget:otherClass];
}
```
打印：message transform third part OtherClass

用途：
- 在方法不能识别的时候做一些保护，防止crash
- 调试的时候打印一些感兴趣的日志
- 也可以hook系统的方法玩玩呀



