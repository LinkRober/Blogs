+++
date = "2017-10-09T18:06:28+08:00"
title = "Runtime (二)  动态方法实现"
draft = true

+++

有的时候，你可能想要提供方法的动态实现。例如，Object-C 属性特征包含了`@dynamic`
```
@dynamic propertyName;
```
这里是告诉编译器属性相关方法的实现会被动态的提供。
你可以实现`resolveInstanceMethod:`和`resolveClassMethod:`为各个实例和类动态的提供一个selector。
<!--more-->

一个Object-C方法是一个简单的C函数，它最少持有两个参数--`self`和`_cmd`。你可以使用` class_addMethod`添加一个函数到类中作为方法，函数大概像这样：
```
void dynamicMethodIMP(id self,SEL _cmd) {
	//implementation...
}
```
你可以动态的把它添加到类中作为一个方法，通过使用`resolveInstanceMethod:`，像这样：
```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
	
}
```


