+++
title = "Weak的实现（三）"
date = "2020-12-08T00:00:00Z"
categories = ["iOS"]
keywords = ["Weak"]
tags = ["Weak"]
draft = false
+++

#### 3 设置弱引用标志位

<!--more-->

```
inline void
objc_object::setWeaklyReferenced_nolock()
{
 retry:
    //去对象的isa指针
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    //如果不是non-pointer
    if (slowpath(!newisa.nonpointer)) {
        ClearExclusive(&isa.bits);
        //<<3.1>>
        sidetable_setWeaklyReferenced_nolock();
        return;
    }
    if (newisa.weakly_referenced) {
        ClearExclusive(&isa.bits);
        return;
    }
    //弱引用标志位设为1
    newisa.weakly_referenced = true;
    //如果oldisa.bits和newisa.bits不相等返回NO，继续tery里面的内容，这时候newisa.weakly_referenced已经是true了，所以return
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}
```
设置nonpointer类型的isa和非nonpointer类型的isa的弱引用位为1


##### 3.1 设置nonpointer的isa指针的弱引用标志位
```
void 
objc_object::sidetable_setWeaklyReferenced_nolock()
{
#if SUPPORT_NONPOINTER_ISA
    ASSERT(!isa.nonpointer);
#endif

    SideTable& table = SideTables()[this];
    table.refcnts[this] |= SIDE_TABLE_WEAKLY_REFERENCED;
}
```
`RefcountMap`，也是哈希表
```
typedef objc::DenseMap<DisguisedPtr<objc_object>,size_t,RefcountMapValuePurgeable> RefcountMap;
```
key是`DisguisedPtr<objc_object>`即`weak_referrer_t`，弱引用对象，value是`size_t`，弱引用数量

这里将`table.refcnts[this]`即最后一位与SIDE_TABLE_WEAKLY_REFERENCED进行位或操作，这时候弱引用标志位变成**1**。


#### 答题
问题1、2、3、5我们放在一起回答

1. 大家都知道weak的底层实现是一个散列表，那么散列表的结构是什么样的？
2. 散列表的key是什么，value是什么，散列函数是怎样的？
3. 通过几次查找才能找到对应的弱引用？
5. 一个对象对应一个`SideTable`表而一个`SideTable`对应多个对象，为什么这样设计

大概的讲是一个散列表`SideTablesMap`，以对象为`key`,`SideTable`为`value`。散列函数是
```
static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
```
但这只是开始，它取到的是很多对象弱引用表的集合。要想准确的找到某个对象的弱引位置用还要经过两步。

- 以对象为key，经过一系列运算（位运算，算数运算），最后通过BITMASK找到`weak_entries`的入口`index`开始变量，判断对象是否相等。最后才能找到对象所在的`weak_entry_t`。
- 因为`weak_entry_t`这里面保存了对象的所有有弱引用，要找到指定的，还要经历和上述类似的操作对比`old_referrer`才能找到正真的弱引用`weak_referrer_t`位置。

所以经过3次查找才能找到正真的弱引用。

为什么苹果要创建64个`SideTable`（在iphone上是8个，其他上面是64个），而不是用一个SideTable解决呢。简单的类比，大家都在火车站或者飞机场打过出租车吧，当你前面的队伍黑压压的一片看不到头你是不是希望多开几个出租车上车点。使用多个`SideTable`也是这个原理提高弱引用的索引速度。

还有一点，我想提下，在整个过程中有两次扩容，一次收缩容量。它们都是动态的。分别是在
- `weak_unregister_no_lock`的时候，收缩`weak_table_t`的容量
```
// Shrink the table if it is mostly empty.
static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Shrink if larger than 1024 buckets and at most 1/16 full.
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
        // leaves new table no more than 1/2 full
    }
}
```
- `append_referrer`的时候增加`weak_entry_t`中动态数组`*referrers`的容量
```
__attribute__((noinline, used))
static void grow_refs_and_insert(weak_entry_t *entry, 
                                 objc_object **new_referrer)
{
    ASSERT(entry->out_of_line());

    size_t old_size = TABLE_SIZE(entry);
    size_t new_size = old_size ? old_size * 2 : 8;

    size_t num_refs = entry->num_refs;
    weak_referrer_t *old_refs = entry->referrers;
    entry->mask = new_size - 1;
    
    entry->referrers = (weak_referrer_t *)
        calloc(TABLE_SIZE(entry), sizeof(weak_referrer_t));
    entry->num_refs = 0;
    entry->max_hash_displacement = 0;
    
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            append_referrer(entry, old_refs[i]);
            num_refs--;
        }
    }
    // Insert
    append_referrer(entry, new_referrer);
    if (old_refs) free(old_refs);
}
```
- `weak_grow_maybe`的时候,增加`weak_table_t`的容量
```
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Grow if at least 3/4 full.
    if (weak_table->num_entries >= old_size * 3 / 4) {
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}
```

问题4

4. 如何查找弱引用对象的引用计数？

```
uintptr_t
_objc_rootRetainCount(id obj)
{
    ASSERT(obj);

    return obj->rootRetainCount();
}

inline uintptr_t 
objc_object::rootRetainCount()
{
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    //获取isa的比特位
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    //是不是non-pointer
    if (bits.nonpointer) {
        //引用比特位上的引用数加1
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            //判断sidetable是否存在引用计数，如果存在继续相加
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}
```
从`rootRetainCount`源码可以看出，弱引用的引用计数分布在两个地方，一个是`isa`的比特位中，还有一个在`SideTable`中。它们相加才是对象的引用计数的数量。


最后祭上Weak的类图和四层存储结构图，结构图中有获取对应层级所用到的算法
![](https://note.youdao.com/yws/api/personal/file/WEBe05b83f81e4040473f0385979441c881?method=download&shareKey=028270cc018dcad1590865e61eb2624c)
![](https://note.youdao.com/yws/api/personal/file/WEB97fdb40aae006524c1682b57022200cd?method=download&shareKey=ec82da0a1b701817110cc4a24494437a)

参考：

[iOS管理对象内存的数据结构以及操作算法--SideTables、RefcountMap、weak_table_t-一
](https://www.jianshu.com/p/ef6d9bf8fe59)

[SideTable结构
](https://www.leewong.cn/2020/08/16/sidetablestructure/)