+++
date = "2017-10-09T18:06:28+08:00"
title = "Runtime (二)  动态方法实现"
categories = ["development"]
keywords = ["runtime"]
tags = ["runtime"]
draft = false

+++

### Dynamic Method Resolution
有的时候，你可能想要提供方法的动态实现。例如，Object-C 属性特征包含了`@dynamic`
<!--more-->
```
@dynamic propertyName;
```
这里是告诉编译器属性相关方法的实现会被动态的提供。

你可以实现`resolveInstanceMethod:`和`resolveClassMethod:`为各个实例和类动态的提供一个selector。

一个Object-C方法是一个简单的C函数，它最少持有两个参数--`self`和`_cmd`。你可以使用` class_addMethod`添加一个函数到类中作为方法，函数大概像这样：
```
void dynamicMethodIMP(id self,SEL _cmd) {
	//implementation...
}
```
你可以动态的把它添加到类中作为一个方法，通过使用`resolveInstanceMethod:`调用`resolveThisMethodDynamically `方法，像这样：
```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
	if(aSEL == @selector(resolveThisMethodDynamically)){
		class_addMethod([self class],aSEL,(IMP)dynamicMethodIMP,"v@:");
		return YES;
	}
	return [super resolveInstanceMethod:aSEL];
}
```
@end
一个类在消息转发机制开始前有机会动态的添加一个方法。如果` respondsToSelector:`或者`instancesRespondToSelector:`方法被调用时，动态的方法接受者首先会被给予一次机会，为selector提供一个`IMP`。如果你实现了` resolveInstanceMethod:`方法，但是想要指定的selector，通过转发机制被转发，你要return `NO`。

### Dynamic Loading
一个Object-C程序可以在运行的时候加载和链接新的类。动态加载可以用来做很多不同的事情。例如，系统偏好中的不同模块是通过动态加载的方式实现的。
在Cocoa的环境中，动态加载一般被用来让应用更具灵活性。在编写自己的模块的时候，可以在runtime的时候被加载。更多的是用接口加载自定义的模版。OS X的系统设置中加载自定义的偏好模块。你应用程序的模块扩展等。
因此在Mach-O文件中有一个runtime函数执行Object-C的动态加载(objc_loadModules,定义在objc/objc-load.h中)，Cocoa的`NSBundle`类为动态加载提供了一个更方便的接口--面向对象的聚合相关服务的接口。


