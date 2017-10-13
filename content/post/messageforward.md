+++
keywords = ["runtime"]
tags = ["runtime"]
categories = ["development"]
date = "2017-10-13T00:00:00Z"
title = "Runtime(三) 消息转发"

+++

### Message Forwarding

向对象发送一个消息，如果没有处理就会发生错误。但是在产生错误之前，runtime会给接受对象第二次机会来处理消息。

### Forwarding

向对象发送一个消息，如果没有处理就会发生错误。但是在产生错误之前，runtime会向对象发送`forwardInvocation:`消息，消息携带了一个`NSInvocation`对象，这个对象里面包裹了原始消息和参数。

你可以实现`forwardInvocation:`方法来处理消息，来避免错误的发生。正如方法名所暗示的`forwardInvocation:`一般用来向另一个对象转发消息。

为了看到转发的过程和结果，我们想象下面的场景：首先，你设计了一个对象能够响应`negotiate`方法，你希望它能够响应另外一个对象的消息。无论`negotiate`的方法体是在哪里实现的,你可以通过传入`negotiate`消息到另一个对象来完成。下一步，假设你想要`negotiate`消息的响应可以被另外一个类明确的接受。一种办法是通过继承的方式，让你的类从另一个类中继承这个方法。但是如果在不同的继承链中，这种方法是不可能的。

即使你的类不能通过继承的方式使用`negotiate`方法，我们仍然可以借用这种方式，向另一个类对象传递消息。

```
- (id)negotiate
{
    if ( [someOtherObject respondsTo:@selector(negotiate)] )
        return [someOtherObject negotiate];
    return self;
}
```
但是这种方式有点笨重，特别是如果你有大量的消息需要传递到另一个类的时候。你必须要在一个方法里面覆盖所有你想要调用的其他类的方法。而且，可能有些方法你不能考虑周全。



<!--more-->
