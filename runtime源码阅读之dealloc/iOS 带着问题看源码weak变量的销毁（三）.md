# iOS 带着问题看源码weak变量的销毁（三）
❓❓❓：超出作用域，weak变量会怎么处理？;

![图片](https://user-images.githubusercontent.com/84235579/118441504-dde98200-b71b-11eb-8d61-2d9074a88cda.png)


接上文，我们知道
```objective-c
int main(int argc, const char * argv[]) {
   {
     Person *person = [Person new];
      __weak Person *weakPerson = person;
     __weak Person *weakPerson2 = person;
   }
    return 0;
}

// 等加于
int main(int argc, const char * argv[]) {
   {
     Person *person = [Person new];
     objc_initWeak(weakPerson, person);
     objc_initWeak(weakPerson2, person);
     objc_destroyWeak(weakPerson2);
     objc_destroyWeak(weakPerson);
     objc_storeStrong(person, nil);
    }
    return 0;
}
```

```objective-c
// 清楚弱引用和弱引用表的关系
void
objc_destroyWeak(id *location)
{
    // 注意这里是DontHaveNew，即只有销毁
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}
```
```objective-c
static id 
storeWeak(id *location, objc_object *newObj)
{
    ASSERT(haveOld  ||  haveNew);
    if (!haveNew) ASSERT(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

 retry:
    if (haveOld) {
        oldObj = *location;
        StripedMap<SideTable>& result = SideTables();
        oldTable = &result[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // 清除旧对象，即对应之前的destroyWeak
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (haveNew) {
        // haveNew = false 这里不再细说
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    callSetWeaklyReferenced((id)newObj);

    return (id)newObj;
}
```

```objective-c
/** 
 * 移除一个弱引用的注册
 * 避免野指针
 * @param weak_table 全局弱引用表.
 * @param referent 弱引用对象.
 * @param referrer 弱引用.
 */
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;

    if (!referent) return;

    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }
        // 如果空了，则将entry移除
        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```

```objective-c
/** 
 * 移除旧的弱引用
 * @param entry 持有该弱引用的entry.
 * @param old_referrer 将要被移除的引用 
 */
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    // 如果是存储在inline_referrers中，置空即可
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

   // // 根据对象地址，hash成索引index, 查找referrers中的弱引用，并置位nil
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

```objective-c
 // 将entry从弱引用表中移除
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
    // remove entry
    if (entry->out_of_line()) free(entry->referrers);
    bzero(entry, sizeof(*entry));

    weak_table->num_entries--;

    weak_compact_maybe(weak_table);
}
```

```objective-c
// 缩减弱引用表的大小
static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // 当 大于1024 且 现有的弱引用entry小于等于 1 / 16 的时候才缩减
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
       // 是弱引用不超过 1 / 2 满
        weak_resize(weak_table, old_size / 8);
    }
}
```
```objective-c
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
    size_t old_size = TABLE_SIZE(weak_table);

    weak_entry_t *old_entries = weak_table->weak_entries;
    // 分配更小的weak_table->weak_entries
    weak_entry_t *new_entries = (weak_entry_t *)
        calloc(new_size, sizeof(weak_entry_t));

    weak_table->mask = new_size - 1;
    weak_table->weak_entries = new_entries;
    weak_table->max_hash_displacement = 0;
    weak_table->num_entries = 0;  // restored by weak_entry_insert below
    
    // 将旧的迁移到新的，更小的当中去，减少内存的占用
    if (old_entries) {
        weak_entry_t *entry;
        weak_entry_t *end = old_entries + old_size;
        for (entry = old_entries; entry < end; entry++) {
            if (entry->referent) {
                weak_entry_insert(weak_table, entry);
            }
        }
        free(old_entries);
    }
}
```



