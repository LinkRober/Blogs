+++
title = "Weak的实现（一）"
date = "2020-12-08T00:00:00Z"
categories = ["iOS"]
keywords = ["Weak"]
tags = ["Weak"]
draft = false
+++

本文较长分三篇按序阅读体验更佳，第四篇为辅助阅读按需看

1. [Weak的实现（一）](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0%E4%B8%80/)
2. [Weak的实现（二）](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0%E4%BA%8C/)
3. [Weak的实现（三）](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0%E4%B8%89/)
4. [Weak的实现-&SideTables()[oldObj]](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0-sidetablesoldobj/)

带着问题看源码：

1. 大家都知道weak的底层实现是一个散列表，那么散列表的结构是什么样的？
2. 散列表的key是什么，value是什么，散列函数是怎样的？
3. 通过几次查找才能找到对应的弱引用？
4. 如何查找弱引用对象的引用计数？
5. 一个对象对应一个`SideTable`表而一个`SideTable`对应多个对象，为什么这样设计

<!--more-->

>散列表：散列表（Hash table），根据键直接方法在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做散列函数，存放记录的数组称做散列表。

>散列函数：它是一个函数，如果把它定义成hash(key)，其中key表示元素的键值，在hash(key)的值表示经过散列函数计算得到的散列值


从代码开始
```
{
    NSObject *obj = [[NSObject alloc] init];
    id __weak obj1 = obj;
}
```
创建`weak`引用的时候会走到`runtime`的`objc_initWeak`这个方法里面。通过符号断点可以验证。

`runtime`里的入口
```
id objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```
可以看到是走到了
```
/*
@param `*location`:weak的指针地址
@param `objc_object`:被引用的对象
*/
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating> 
storeWeak(id *location, objc_object *newObj)

{
    //校验旧对象和新对象必须存其一
    ASSERT(haveOld  ||  haveNew);
    //校验如果haveNew=true，newObj不能为nil
    if (!haveNew) ASSERT(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) {
    //如果weak ptr存在旧值，就取出旧值
        oldObj = *location;
        //以旧对象为析构函数的入参取出旧的SideTable
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
     //如果weak ptr是新值，以新对象为析构函数的入参取出对应的SideTable
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    
    //将oldTable和newTable都上锁
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
    //校验，如果旧值对不上 goto retry
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    //保证弱引用对象的isa非空，防止弱引用机制和+initialize 发生死锁
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            //如果class没有初始化发送+initialized消息
            class_initialize(cls, (id)newObj);

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;
            //到这里class肯定已经初始化了，在走一遍
            goto retry;
        }
    }

    // Clean up old value, if any.
    // 如果weak ptr之前引用了其他对象，在这里清空
    if (haveOld) {
    //<<1>>
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (haveNew) {
    //通过newObj和location生成一个新的weak_entry_t并插入到newObj的弱引用数组中（weak_entries）
    //<<2>>
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
           //<<3>> 
           newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```
- 获取对象所在的`SideTable`
- `isa`非空校验，如果`isa`没有初始化执行`class_initialize(cls, (id)newObj);`方法
- 如果地址`location`的引用对象已经存在，删除其在`weak_table_t`表中的所有引用
- 注册新对象的弱引用到`weak_table_t`表中
- 设置新对象的弱引用标志符为`YES`


如果对`&SideTables()[oldObj]`不太理解的可以先移步[这篇文章](http://note.youdao.com/s/dIRTiheA)
>阅读提醒：每一个<<>>中的序号后面都会扩展开讲

#### 1.清除老对象的弱引表

```
/** 
 * @param weak_table 某个对象的全局weak_table.
 * @param referent 当前对象
 * @param referrer 当前对象的弱引用地址
 */
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;//二级指针 weak_ptr的地址

    weak_entry_t *entry;

    if (!referent) return;
    //<<1.1>>查找referent对应的weak_entry_t
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        //<<1.2>>如果entry存在，删除entry
        remove_referrer(entry, referrer);
        bool empty = true;
        //判断entry的动态数组referrers中是否有值
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            //判断entry的定长数组inline_referrers中是否有值
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }
        //如果都是空的将entry从weak_table移除
        if (empty) {
            //<<1.3>>
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
```
该方法的主要目的是清除存储在`entry`中的`weak_referrer_t`，如果发现`entry`中一个`weak_referrer_t`也没有，就将整个`entry`从`weak_table`中移除。

##### 1.1查找`referent`对应的`entry`
```
* @param weak_table 对象的weak_table 
* @param referent 当前对象
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    ASSERT(referent);

    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
```
```
size_t begin = hash_pointer(referent) & weak_table->mask;
```
- 通过指针的哈希方法生成的值与`weak_table->mask`进行[BITMASK](https://en.wikipedia.org/wiki/Mask_(computing))操作得到一个起始值，这个等下会提一下。

```
#if __LP64__
static inline uint32_t ptr_hash(uint64_t key)
{
    key ^= key >> 4;
    key *= 0x8a970be7488fda55;
    key ^= __builtin_bswap64(key);
    return (uint32_t)key;
}
#else
static inline uint32_t ptr_hash(uint32_t key)
{
    key ^= key >> 4;
    key *= 0x5052acdb;
    key ^= __builtin_bswap32(key);
    return key;
}
#endif

```

- 每次遍历如果没在`weak_entries`中找到`referent`就对index加1再进行BITMASK操作。遍历一次就认为是哈希冲突一次并记录在遍历`hash_displacement`
- 如果哈希冲突超过了最大值返回nil，当前对象在`weak_table`中不存在弱引用
- 成功找到对应的`referent`就返回相应的`weak_entry_t`


简单说一下**BITMASK**技术:</br>

`weak`的实现中在对`weak_entry_t`和`weak_referrer_t`的遍历查找都是通过`BITMASK`来实现的。个人猜测先经过一系列运算得到底位是表示数量的二进制数，在进行BITMASK操作。

在日常开发中用的不多，主要是用于对二进制位进行操作。能快速的得到我们只关心位的值。

|value|位操作|mask|结果|
|-|:-:|-|:-:|
|0x00000000|&|0x000000011|0|
|0x00000001|&|0x000000011|1|
|0x00000010|&|0x000000011|2|
|0x00000011|&|0x000000011|3|
|0x00000100|&|0x000000011|0|

可以看到如果我们只关心低两位地址，进行BITMASK就能屏蔽其他位。所以上文中的`weak_table->mask`就是`weak_table_t`中`weak_entry_t`的最大容量减1（从0开始算）。从`static void weak_resize(weak_table_t *weak_table, size_t new_size)`方法中可以看出
```
...
weak_table->mask = new_size - 1;
...
```

#### 理解结构体`weak_entry_t`
在继续下去之前先看下这个结构体，要注意的是它里面有个联合体，联合体里面的两个结构体共享内存。而他们的区别在于，如果弱引用的数量不大于4个就有定长数组`inline_referrers`，否则使用动态数组`referrers`。
```
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;//弱引用对象
    union {//联合体，两种结构体共占有一块内存
        //弱引用数量大于4个用到的结构体
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        //弱引用数量不大于4个用到的结构体
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
    
    //判断是否是用的referrers来存储弱引用指针
    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }
    //覆盖老数据
    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }
    //构造方法
    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```
这个里面存放了某个对象的所有弱引用指针，如果弱引用对象数量不超过四个就报错在结构体数组`inline_referrers`，否则保存在`referrers`

##### 1.2删除 `entry`中的`referrer`
```
/** 
 * @param entry 当前对象对应的weak_entry_t
 * @param old_referrer 弱引用指针地址
 */
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    if (! entry->out_of_line()) {
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == old_referrer) {
                entry->inline_referrers[i] = nil;
                return;
            }
        }
        _objc_inform("Attempted to unregister unknown __weak variable "
                     "at %p. This is probably incorrect use of "
                     "objc_storeWeak() and objc_loadWeak(). "
                     "Break on objc_weak_error to debug.\n", 
                     old_referrer);
        objc_weak_error();
        return;
    }

    size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != old_referrer) {
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
        hash_displacement++;
        if (hash_displacement > entry->max_hash_displacement) {
            _objc_inform("Attempted to unregister unknown __weak variable "
                         "at %p. This is probably incorrect use of "
                         "objc_storeWeak() and objc_loadWeak(). "
                         "Break on objc_weak_error to debug.\n", 
                         old_referrer);
            objc_weak_error();
            return;
        }
    }
    entry->referrers[index] = nil;
    entry->num_refs--;
}
```
- 如果定长数组`inline_referrers`中有值且存在弱引用指针`old_referrer`，设为nil
- 如果动态数组`referrers`中有值且存在弱引用指针`old_referrer`，设为nil,并将引用数量-1

##### 1.3 如果有必要删除对象的整个弱引用表
```
**
 * Remove entry from the zone's table of weak references.
 */
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
    // remove entry
    if (entry->out_of_line()) free(entry->referrers);
    bzero(entry, sizeof(*entry));

    weak_table->num_entries--;

    weak_compact_maybe(weak_table);
}
```
- 如果使用的是动态数组，释放动态数组的内存
- 以`entry`为起始地址的前`sizeof(*entry)`个字节区域清零
- 全局`weak_table`中，弱引用对象数量-1
- 收缩表大小
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
如果`weak_table`内存占用超过**1024**字节且内存的1/16比弱引用对象的数量还多就收缩表大小，使其不大于原来的**1/2**。

