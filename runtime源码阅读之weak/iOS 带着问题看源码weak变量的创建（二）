# iOS 带着问题看源码weak变量的创建（二）

❓❓❓：weak变量是怎么添加到weak表中的？
前情概要：
![图片]()

weak变量的存储过程：
![图片]()

```objective-c
int main(int argc, const char * argv[]) {
    Person *person = [Person new];
    __weak Person *weakPerson = person;
    __weak Person *weakPerson2 = person;
    
    return 0;
}

@interface Person : NSObject
@end
@implementation Person
@end
```
clang编译后的中间语言
```objective-c
define i32 @main(i32 %0, i8** %1) #1 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i8**, align 8
  %6 = alloca %0*, align 8
  %7 = alloca %0*, align 8
  %8 = alloca %0*, align 8
  store i32 0, i32* %3, align 4
  store i32 %0, i32* %4, align 4
  store i8** %1, i8*** %5, align 8
  // Person *person = [Person new];
  %9 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %10 = bitcast %struct._class_t* %9 to i8*
  %11 = call i8* @objc_opt_new(i8* %10)
  %12 = bitcast i8* %11 to %0*
  store %0* %12, %0** %6, align 8
  %13 = load %0*, %0** %6, align 8
  %14 = bitcast %0** %7 to i8**
  %15 = bitcast %0* %13 to i8*
  // objc_initWeak(weakPerson, person);
  %16 = call i8* @llvm.objc.initWeak(i8** %14, i8* %15) #2
  %17 = load %0*, %0** %6, align 8
  %18 = bitcast %0** %8 to i8**
  %19 = bitcast %0* %17 to i8*
  // objc_initWeak(weakPerson2, person);
  %20 = call i8* @llvm.objc.initWeak(i8** %18, i8* %19) #2
  store i32 0, i32* %3, align 4
  %21 = bitcast %0** %8 to i8**
  // objc_destroyWeak(weakPerson2);
  call void @llvm.objc.destroyWeak(i8** %21) #2
  %22 = bitcast %0** %7 to i8**
  // objc_destroyWeak(weakPerson);
  call void @llvm.objc.destroyWeak(i8** %22) #2
  %23 = bitcast %0** %6 to i8**
  // objc_storeStrong(person, nil);
  call void @llvm.objc.storeStrong(i8** %23, i8* null) #2
  %24 = load i32, i32* %3, align 4
  ret i32 %24
}

// 等加于
int main(int argc, const char * argv[]) {
    Person *person = [Person new];
   
    objc_initWeak(weakPerson, person);
    objc_initWeak(weakPerson2, person);
    
    objc_destroyWeak(weakPerson2);
    objc_destroyWeak(weakPerson);
    
    objc_storeStrong(person, nil);
    return 0;
}
```

```objective-c
/** 
 * 初始化一个弱引用指针
 * 例如这样: 
 *
 * (The nil case) 
 * __weak id weakPtr;
 * (The non-nil case) 
 * NSObject *o = ...;
 * __weak id weakPtr = o;
 * 初始化一个weak指针，并行修改弱引用变量是非线程安全的
 */
id
objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```
继续看之前先回顾一下后面文章讲的，weak变量的存储结构
对象的弱引用 比喻成 学生拥有的书

|数据结构 | 类比 |
|---|---|
|NSObject * | 1个学生 |
|StripedMap | 1个教学楼，N个教室 |
|SideTable | 1个教室：包括一把锁（分离锁）|
|weak_table_t | N个学生（同一个教室） |
|weak_entry_t | 1个学生的N本书 |
|weak_referrer_t | 1本书 | |

```objective-c
// 更新弱引用变量
// HaveOld == true， 则变量之前指向的对象需要被清理
// HaveNew == true, 则新对象会被赋值给weak变量
// CrashIfDeallocating = true, 如果新对象正在释放或者newObj's class不支持弱引用，则程序挂掉，否则赋值为nil
enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};
template <HaveOld haveOld, HaveNew haveNew,
          enum CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    ASSERT(haveOld  ||  haveNew);
    if (!haveNew) ASSERT(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    // weak变量之前指向的对象对应的弱引用表 
    SideTable *oldTable;
    // weak变量新对象对应的弱引用表
    SideTable *newTable;
    // 两个sideTable有可能是同一个sideTable，也有可能，就像两个学生可能来自同一个宿舍，也有可能来自两个宿舍

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    // 根据对象指针查出对应的弱引用表，对象地址hash成数组index
    // int index = ((addr >> 4) ^ (addr >> 9)) % StripeCount;  StriperCount = 64;
    // 就像根据学生学号查找对应的宿舍
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
    // 对这两个宿舍上锁，不要让其他人访问和修改
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
     // 如果haveOld = true 但弱引用指向的变量不是旧对象，在查表之后被修改了，则重新查表
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // 略掉特殊情况处理
    
    // Clean up old value, if any. 清除指向的旧对象中弱引用表中的弱引用，细节见后文weak变量的销毁
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any. 指定新对象
    if (haveNew) {
        // 将弱引用添加到弱引用表
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating ? CrashIfDeallocating : ReturnNilIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        // newisa.weakly_referenced = true;
        // 修改弱引用在对象isa中的相关存储，细节见后文
        if (!newObj->isTaggedPointerOrNil()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    // This must be called without the locks held, as it can invoke
    // arbitrary code. In particular, even if _setWeaklyReferenced
    // is not implemented, resolveInstanceMethod: may be, and may
    // call back into the weak reference machinery.
    callSetWeaklyReferenced((id)newObj);

    return (id)newObj;
}
```

```objective-c
/**
 * 将对象和弱引用指针注册到弱引用表，如果第一次
 * 则创建object entry
 * @param weak_table 全局的弱引用表
 * @param referent 弱引用指向的对象
 * @param referrer 弱引用地址
 */
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, WeakRegisterDeallocatingOptions deallocatingOptions)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (referent->isTaggedPointerOrNil()) return referent_id;

    // 确保引用对象是存活的 deallocatingOptions == ReturnNilIfDeallocating || deallocatingOptions == CrashIfDeallocating

    // 存储对象的弱引用，并记录存在哪
    // 如果第一次存储，则创建该对象的object entry, 即对应一个对象的所有若应用，一个学生拥有的所有书
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}

```

```objective-c
/** 
 * 将一个新创建的object entry 添加到弱引用表中，并不会检查是否插入过了，在外部检查
 */
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;
    ASSERT(weak_entries != nil);

    // 根据对象地址，hash成索引index
    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    // 如果该处已经有对象了，则index + 1,并记录对应的偏移位置
    while (weak_entries[index].referent != nil) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }

    // 设置entry
    weak_entries[index] = *new_entry;
    weak_table->num_entries++;

// 修改弱引用表的max_hash_displacement
    if (hash_displacement > weak_table->max_hash_displacement) {
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```

```objective-c
/** 
 *将给定的弱引用添加到已有的object entry中
 *
 * @param entry 持有弱引用指针的entry
 * @param new_referrer 需要被添加的弱引用
 */
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
          // 尝试现将弱引用添加到inline_referrers中
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
        // 如果还有空的话，则添加
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }

        // 不能插入到inline_referrers中，则分配outline 即entry->referrers的空间
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        // 现在表的结构是无效的，grow_refs_and_insert会处理它 （重新hash）
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

    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        return grow_refs_and_insert(entry, new_referrer);
    }
    // 根据对象地址，hash成索引index
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    // 如果该处已经有弱引用了，则index + 1,并记录对应的偏移位置
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    // 设置弱引用
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```

```objective-c
/** 
 * 扩增 object entry的hast table ,并重新hash每个弱引用
 */
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
    // 之前的重新hash
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            append_referrer(entry, old_refs[i]);
            num_refs--;
        }
    }
    // 插入新增的弱引用
    append_referrer(entry, new_referrer);
    if (old_refs) free(old_refs);
}
```
至此，weak变量存储完成。



