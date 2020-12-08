+++
title = "Weak的实现（二）"
date = "2020-12-08T00:00:00Z"
categories = ["iOS"]
keywords = ["Weak"]
tags = ["Weak"]
draft = false
+++

1. [Weak的实现（一）](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0%E4%B8%80/)
2. [Weak的实现（二）](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0%E4%BA%8C/)
3. [Weak的实现（三）](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0%E4%B8%89/)
4. [Weak的实现-&SideTables()[oldObj]](https://linkrober.github.io/bookshelf/2020/12/weak%E7%9A%84%E5%AE%9E%E7%8E%B0-sidetablesoldobj/)

##### 正文 接 Weak的实现（一） 

#### 2 生成新的`weak_entry_t`插入到`weak_entries`中

<!--more-->

```
/** 
 * Registers a new (object, weak pointer) pair. Creates a new weak
 * object entry if it does not exist.
 * 
 * @param weak_table The global weak table.弱引用全局表
 * @param referent The object pointed to by the weak reference. 弱引用对象
 * @param referrer The weak pointer address. 弱引用指针地址
 */
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    // 判断对象是否正在释放或者是否支持弱引用
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           @selector(allowsWeakReference));
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, @selector(allowsWeakReference));
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    // 如果对象已经在weak_table中存在弱引用记录，就原来的entry上面追加
    weak_entry_t *entry;
    //<<1.1>>
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
    //<<2.1>>
        append_referrer(entry, referrer);
    } 
    else {
        //创建新的entry，添加到weak_table中
        //<<2.2>>
        weak_entry_t new_entry(referent, referrer);
        //<<2.3>>
        weak_grow_maybe(weak_table);
        //<<2.4>>
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```

##### 2.1 在`weak_entry_t`添加新的弱引用`weak_referrer_t`
```
/** 
 * Add the given referrer to set of weak pointers in this entry.
 * Does not perform duplicate checking (b/c weak pointers are never
 * added to a set twice). 
 *
 * @param entry The entry holding the set of weak pointers. 某个类的弱引用表
 * @param new_referrer The new weak pointer to be added.新的弱引用
 */
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    //如果inline_referrers中还能存放weak_referrer_t就放在inline_referrers里面
    if (! entry->out_of_line()) {
        // Try to insert inline.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }

        // Couldn't insert inline. Allocate out of line.
        // 如果放不下了，就创建把所有的weak_referrer_t挪到referrers中
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        // This constructed table is invalid, but grow_refs_and_insert
        // will fix it and rehash it.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[i];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }

    ASSERT(entry->out_of_line());
    // 如果引用数量超过表内存的3/4就自动扩容
    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        //<<2.1.1>>
        return grow_refs_and_insert(entry, new_referrer);
    }
    //在referrers中找到一个值为nil的weak_referrer_t对象，用新的弱引用对其赋值，并自增数量
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```
如果`weak_table_t`中存在当前对象的弱引用记录`weak_entry_t`，使用该方法在`weak_entry_t`添加新的`weak_referrer_t`。
- 如果`weak_entry_t`中的`weak_referrer_t`数量不超过4个，即`weak_referrer_t`用还保存在`inline_referrers`中，就将其添加到`inline_referrers`中；
- 如果刚好满4个，就将所有的`weak_referrer_t`从定长数组`inline_referrers`中取出，开辟新的空间，存储到动态数组`referrers`中来记录引用
- 如果`weak_referrer_t`的数量大于表内存大小的3/4，自动扩容<<2.2.1>>
- 在`referrers`找到第一个为nil的`weak_referrer_t`指针，新的`weak_referrer_t`赋值给它，引用计数自增。


##### 2.1.1 增加某个对象弱引用表的容量

```
/** 
 * Grow the entry's hash table of referrers. Rehashes each
 * of the referrers.
 * 
 * @param entry Weak pointer hash set for a particular object.某个对象的弱引用表
 */
__attribute__((noinline, used))
static void grow_refs_and_insert(weak_entry_t *entry, 
                                 objc_object **new_referrer)
{
    ASSERT(entry->out_of_line());
    //原来表大小
    size_t old_size = TABLE_SIZE(entry);
    //新表大小，old_size大于0，新size就变成原来大小的两倍，否则就8个字节大小
    size_t new_size = old_size ? old_size * 2 : 8;

    //原来的弱引用数量
    size_t num_refs = entry->num_refs;
    //原来的弱引用数组
    weak_referrer_t *old_refs = entry->referrers;
    //新的mask
    entry->mask = new_size - 1;
    
    //重新分配entry的内存
    entry->referrers = (weak_referrer_t *)
        calloc(TABLE_SIZE(entry), sizeof(weak_referrer_t));
    entry->num_refs = 0;
    entry->max_hash_displacement = 0;
    
    //先将老的引用全部插入到新的数组里面
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            //递归调用 插入weak_referrer_t
            append_referrer(entry, old_refs[i]);
            num_refs--;
        }
    }
    // Insert
    // 最后插入新加入的弱引用
    append_referrer(entry, new_referrer);
    if (old_refs) free(old_refs);
}
```
此方法重新定义了对象弱引用表`weak_entry_t`的大小
1. 原来的表大小等于0，就分配8个字节
2. 如果原来表大小大于0，就分配原来大小2倍的字节
3. 然后为`referrers`重新分配内存，先将老的`weak_entry_t`迁移过去，再插入新的`weak_entry_t`。通过递归调用`append_referrer`实现插入数据


##### 2.2 创建新的`weak_entry_t`
```
    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
```
如果对象还没有弱引用表`weak_entry_t`就该对象建一个新的，从源码可看出`newReferrer`先保存在`inline_referrers`中

##### 2.3 `weak_table_t`的扩容
```
// Grow the given zone's table of weak references if it is full.
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Grow if at least 3/4 full.
    if (weak_table->num_entries >= old_size * 3 / 4) {
        //<<2.3.1>>
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}
```
如果`weak_table`中`num_entries`数组的数量大于其内存大小的3/4就开始为`weak_table`扩容
 - oldsize大于0，扩容为原来大小的两倍
 - 否则分配64字节

##### 2.3.1 weak_resize
```
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
    size_t old_size = TABLE_SIZE(weak_table);

    weak_entry_t *old_entries = weak_table->weak_entries;
    weak_entry_t *new_entries = (weak_entry_t *)
        calloc(new_size, sizeof(weak_entry_t));

    weak_table->mask = new_size - 1;
    weak_table->weak_entries = new_entries;
    weak_table->max_hash_displacement = 0;
    weak_table->num_entries = 0;  // restored by weak_entry_insert below
    
    if (old_entries) {
        weak_entry_t *entry;
        weak_entry_t *end = old_entries + old_size;
        for (entry = old_entries; entry < end; entry++) {
            if (entry->referent) {
                //<<2.3.1.1>>
                weak_entry_insert(weak_table, entry);
            }
        }
        free(old_entries);
    }
}
```
- 创建新的指针数组`new_entries`，分配新的大小
- 遍历将老的数据迁移到新的数组中


##### 2.3.1.1 向`weak_table_t`中插入`weak_entry_t`
```
/** 
 * Add new_entry to the object's table of weak references.
 * Does not check whether the referent is already in the table.
 */
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;
    ASSERT(weak_entries != nil);

    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }

    weak_entries[index] = *new_entry;
    weak_table->num_entries++;

    if (hash_displacement > weak_table->max_hash_displacement) {
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```
还是通过BITMASK来遍历所有的`weak_entry_t`，直到有一个`referent`为nil,就将新的`weak_entry_t`赋值给它。

##### 2.4 将对象的新`weak_entry_t`插入到`weak_table_t`中
```
/** 
 * Add new_entry to the object's table of weak references.
 * Does not check whether the referent is already in the table.
 */
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;
    ASSERT(weak_entries != nil);
    //遍历weak_entries，直到weak_entries[index].referent为nil
    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }
    //将新的数据赋值给referent为nil的指针
    weak_entries[index] = *new_entry;
    //weak_entry_t数量+1
    weak_table->num_entries++;
    //更新max_hash_displacement
    if (hash_displacement > weak_table->max_hash_displacement) {
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```
- 遍历`weak_table_t`中的`weak_entries`，直到有一个`weak_entry_t`的为`referent`为`nil`
- 将新的`weak_entry_t`赋值给`referent`为`nil`的`weak_entry_t`
- 更新`num_entries`和`max_hash_displacement`的数量