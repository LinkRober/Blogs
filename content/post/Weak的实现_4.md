+++
title = "Weak的实现-&SideTables()[oldObj]"
date = "2020-12-08T00:00:00Z"
categories = ["iOS"]
keywords = ["Weak"]
tags = ["Weak"]
draft = false
+++

`&SideTables()[oldObj]`这是什么？很多人看到这里都被这操作搞蒙了，下面分三步来理解，分别是`SideTables()`、`[oldObj]`、`&`。先贴上入口的代码
```
id oldObj;
SideTable *oldTable;

oldObj = *location;
oldTable = &SideTables()[oldObj];
```

<!--more-->


#### 1. 理解`SideTables()`

首先`SideTables()`是一个静态函数，完全体是这样的
```
static StripedMap<SideTable>& SideTables() {
    return SideTablesMap.get();
}
```
函数体里面调用了一个全局的静态变量`SideTablesMap`的`get()`方法，
静态变量里面保存了所有的`SideTable`。的声明如下：
```
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;
```

可以看出`SideTablesMap`是在命名空间objc下面的一个`ExplicitInit`类，它里面实现了`get()`方法，如下
```
Type &get() {
    return *reinterpret_cast<Type *>(_storage);
}
```
对于不熟悉c++的理解下面这几点基本就能看懂：

- c++ 中的引用`&`，在c\+\+中`&`除了有`取地址`的作用还可以作为`引用`。方法返回的是一个引用，如果没有c++基础的很容易误认为返回了一个指针。
- c++中的模板`template`，泛型`Type`是通过模板传入的，即`StripedMap`。
从上面声明`SideTablesMap`的地方可以看到，这里是个模板的嵌套，`StripedMap`是`ExplicitInit`模板的泛型；`SideTable`是`StripedMap`模板的泛型。
- c++中的`reinterpret_cast<new_type>(expression)
`，它可以将两种任何类型进行转换，新值和旧值（expression）有相同的比特位。

最后用`*`取转换后得到的地址的所保存的值，返回，所以说这里返回的不是指针。
`Type *`就相当于`StripedMap *`，所以`get()`方法返回的就是`StripedMap`结构体实例。



#### 2. 理解`[oldObj]`
这里的`[]`其实是对`[]`进行了重载
```
T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
}
```
在`indexForPointer()`方法里，先将结构体指针`oldObj`转化成和其有相同比特位的地址，再进行位移、异或、去模操作得到一个从0-63位之间的`index`，通过`index`，拿到数组里面的`PaddedT`结构体，返回该结构体的`value`成员即结构体`SideTable`。
```
static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
```
```
 struct PaddedT {
        T value alignas(CacheLineSize);
    };
```

其实`StripedMap`是一个以`void *p`为`key`，`PaddedT`为`value`的的表。

#### 3. `&`
最后对`SideTables()[oldObj]`即`SideTable`取地址，这样就和`SideTable *newTable;`对上了。