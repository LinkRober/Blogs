+++
categories = ["runtime"]
date = "2017-09-29T00:00:00Z"
title = "Runtime (一)"
keywords = ["runtime"]
tags = ["runtime"]
draft = true
+++

<!--more-->
### Message
这篇文章描述了消息如何通过使用`objc_msgSend`发送，如何通过方法名称找到对应方法的reference

#### The objc_msgSend Function
在Object-C中，发送的消息直到运行时才绑定到正真的方式实现。
```
[receiver message]
```
`objc_msgSend`是真正调用消息的函数，这个函数持有消息的接受者和方法的名称：
```
objc_msgSend(receiver,selector)
```
任何其他的参数也是通过这个方法传入：
```
objc_msgSend(receiver,selector,arg1,arg2,...)
```
这个方法会做一些必要的事情来进行动态绑定：

- 首先找到方法的实现即selector的引用。因为在不同的类中都可以实现相同方法名的方法，所以通过方法的实现找到真正的receiver的class。
- 然后传入该方法指定的receiver和参数。
- 最后将实现的返回值作为它自己的返回值。

>Note:编译器生成调用消息的方法，在日常开发中永远不要在你写的代码中直接调用。

在架构中消息传递的关键在于编译器编译每个class和object。每个class的结构中包含两个重要元素：

- 指向父类的指针
- 一个类的派发表 ，这个表记录了方法的selector和定义该方法的地址。`setOrigin::`方法和其对应的地址；`display`和其对应的地址。

当一个新的对象被创建，内存被分配，实例变量被初始化。第一件事就是对象变量指针指向它的类结构。这个指针一般被称作`isa`,给予对象访问类的权利。所有的类都是从这个类继承而来。

>Note:尽管这是oc不太严谨的一部分，但是这是oc对象在运行时不可缺少的一部分。一个对象需要一个和它等效的结构体来代表，在任何地方都可以用它来定义。如果你曾经需要创建一个根对象。这个对象自动继承`NSObject`或者`NSProxy`都会携带一个`isa`变量。

类的这些元素和对象结构如下：

![messaging1](../../../messaging1.png)

当消息被发送到一个对象，消息方法跟随对象的`isa`指向类结构，在派发表中一直向上寻找对应的方法。如果你不能在这里找到你需要的方法，`objc_msgSend`方法会继续向上在父类的派发表里面尝试寻找该方法。还是失败就会一直寻找直到到达`NSObject`类，一旦找到selector，就会从表中的入口调用该方法，把它传递到对象的数据结构中。

这种方式的方法调用是在runtime的时候。在面向对象编程的专业术语中，我们称它为—**动态消息绑定**





