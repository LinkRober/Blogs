+++
keywords = ["runtime"]
tags = ["runtime","ObjectC"]
categories = ["iOS"]
date = "2017-10-16T00:00:00Z"
title = "Runtime(三) 消息转发"
draft = false
+++


向对象发送一个消息，如果没有处理就会发生错误。但是在产生错误之前，runtime会给接受对象第二次机会来处理消息。
<!--more-->
### Forwarding

向对象发送一个消息，如果没有处理就会发生错误。但是在产生错误之前，runtime会向对象发送`forwardInvocation:`消息，消息携带了一个`NSInvocation`对象，这个对象里面包裹了原始消息和参数。

你可以实现`forwardInvocation:`方法来处理消息，来避免错误的发生。正如方法名所暗示的`forwardInvocation:`一般用来向另一个对象转发消息。

为了看到转发的过程和结果，我们想象下面的场景：首先，你设计了一个对象能够响应`negotiate`方法，你希望它能够响应另外一个对象的消息。无论`negotiate`的方法体是在哪里实现的,你可以通过传入`negotiate`消息到另一个对象来完成。下一步，假设你想要`negotiate`消息的响应可以被另外一个类明确的接受。一种办法是通过继承的方式，让你的类从父类中继承这个方法。但是如果在不同的继承链中，这种方法是不可能实现的。

即使你的类不能通过继承的方式使用`negotiate`方法，我们仍然可以借用下面这种方式，向另一个类对象传递消息。

```
- (id)negotiate
{
    if ( [someOtherObject respondsTo:@selector(negotiate)] )
        return [someOtherObject negotiate];
    return self;
}
```
但是这种方式有点笨重，特别是如果你有大量的消息需要传递到另一个类的时候。你必须要在一个方法里面覆盖所有你想要调用的其他类的方法。而且，可能有些方法你不能考虑周全。所有的方法集合都依赖于运行时事件，在以后类的实现中作为新的方法发生改变。

第二次机会被`forwardInvocation:`方法所提供，这是一种比较特殊的解决方式，它是动态的而不是静态的。它工作起来就像是：当一个对象不能响应一个消息，因为在消息中没有selector匹配这个方法。runtime会用`forwardInvocation: `消息来通知对象。每个对象都从NSObject里继承了`forwardInvocation:`方法。但是NSObject只是简单的调用`doesNotRecognizeSelector:`。通过重写NSObject版本的实现，你可以利用这次机会，通过`forwardInvocation:`消息提供一个方法转发到另一个类中。
为了转发一个消息，所有的`forwardInvocation:`方法必须包含下面的两个步骤:

- 确定消息的去向
- 转发的时候携带原始的参数

消息可以通过`invokeWithTarget:`方法被发送：
```
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
	if ([someOtherObject respondsToSelector:[anInvocation selector]])
		[aInvocation invokeWithTarget:someOtherObject];
	else
		[super forwardInvocation:anInvocation];
}
```
被转发的消息的返回值被返回到原始的sender。所有返回值的类型包括：ids、structures，双精度浮点数。
对于unrecognized消息，`forwardInvocation:`方法就像一个消息转发中心，将message包裹起来发送到不同的接受者。或者作为一个中转站，发送所有消息到同样的地方。它可以转发一个消息到另一个。`forwardInvocation:`方法可以定位几个消息消息到单一的response。`forwardInvocation:`要做的是准备实现。但是，这次机会为转发链中的链接对象提供更多程序设计的可能。

>**Note:**`forwardInvocation:`方法开始处理消息，只有它们不能调用在接受者中已经存在的方法时。比如，你想要你的对象去转发`negotiate`消息到另一个对象，但对象并没有自己对该消息的实现。如果是这样，消息将永远不会到达`forwardInvocation:`

### Forwarding and Multiple Inheritance

转发可以模仿多继承，可以实现多继承的一些效果。对象通过转发来响应消息，就像是借用或者继承了另一个类的某个方法的实现
![forwarding](../../../forwarding.png)

在这个例子中，Warrior类的实例转发一个`negotiate`消息到Diplomat类的实例。Warrior可以像Diplomat一样调用negotiate。看起来就像响应`negotiate`方法。
对象转发消息，这样“继承”的方法来自两种继承架构链。上面的例子中，它就像Warrior的类继承自Diplomat ，就像子类继承父类一样。

转发提供了很多特性，一般你可以用它来实现多继承。但是，这和多继承有两个很大不同地方：首先，在一个类中，多继承组合出不同的能力。它趋向于较庞大的多接口对象。另一方面，继承可以区分不同种类的方法到不同的对象。

### Forwarding and Inheritance
尽管转发可以模仿继承，NSObject类绝不能混合这两种方式。像`respondsToSelector:`和`isKindOfClass:`方法，只能在继承链中看到，不能用于消息转发链。例如，一个Warrior对象被查询是否响应`negotiate`消息，
```
if([aWarrior respondsToSelector:@selector(negotiate)])
...
```
答案是`NO`，在某种意义上，即使它能无错误的接受`negotiate`消息，并且响应它们，通过转发它们到Diplomat。

在许多情况下，`NO`是正确的答案。但也需不是。如果你使用转发来设置一个替代的object或者扩展一个类的能力，转发机制应该像继承一样透明。如果你想要你的对象的行为就好像真正是通过继承来实现消息转发的。你将需要重新实现`respondsToSelector:`和`isKindOfClass:`包括你的转发算法：
```
- (BOOL)respondsToSelector:(SEL)aSelector
{
	if([super respondsToSelector:aSelector])
		return YESl
	else {
		/* Here,test whether the aSelector message can *
		 * be forwarded to another object and whether that *
		 * object can respond to it.Return YES if it can. */
	}
	return NO;
}
```
除了`respondsToSelector:`和`isKindOfClass:`,`instancesRespondToSelector:`方法应该反映转发算法。如果协议被使用,`conformsToProtocol:`方法应该同样被添加到这个list中。类似的，如果一个对象转发任何它接受的远程消息，它应该有一个`methodSignatureForSelector:`方法的实现，它能返回方法精确的描述。最终响应转发的消息。例如，如果一个对象能够转发一个消息到它的替代者，你应该向下面这样实现`methodSignatureForSelector:`方法：

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector
{
	NSMethodSignature *signature = [super methodSignatureForSelector:selector];
	if(!signature){
		signature = [surrogate methodSignatureForSelector:selector];
	}
	return signature;
}
```
你可能考虑把转发算法放在私有代码的某个地方并且拥有所有这些方法,`forwardInvocation:`包括调用它。

>**Note:**这是一个高级技巧，仅仅适用于没有其他解决方案的时候。这也不将作为继承的替代。如果你必须使用这个技术，确保你能完全理解在进行转发和要转发到的目标类的行为。

在这部分提到的方法被描述在`NSObject`类的说明中。更多的关于`invokeWithTarget:`请看[NSInvocation](https://developer.apple.com/documentation/foundation/nsinvocation)类的说明。





