

# 详细分析objc_msgSend

> 文章中的代码都是在 `x86_64` 架构下运行的，arm架构的代码大同小异。
>
> 这里分析的版本是objc4-706

## 写在前面

上一篇文章分析了在当前类的方法列表中的查找过程，本篇将继续向下分析。详细代码参见于[coding](https://git.coding.net/terrylxt/SuningEBuy.git)

## 链接上一篇文章

```objectivec
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    Method meth;

    // ... 上一篇已介绍过的部分，这里不再赘述
 retry:
    runtimeLock.read(); // 查找方法，所以设置读锁

    // Try this class's cache.
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.
    // 在当前类即分类中查找方法（不往继承链上找）
    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        // 如果找到了，则将方法的实现加入缓存中
        log_and_fill_cache(cls, meth->imp, sel, inst, cls);
        imp = meth->imp;
        goto done;
    }

   // ... 后面其他

 done:
    runtimeLock.unlockRead();

    return imp;
}
```

如果在当前类中，查找到了我们我找的方法`getMethodNoSuper_nolock(cls, sel)`,则会将将方法添加到缓存中，提高下次访问的效率。

`log_and_fill_cache`正如其名，会添加日志输出并调用cache_fill方法，这里直接过

```objectivec
/***********************************************************************
* log_and_fill_cache
* Log this method call. If the logger permits it, fill the method cache.
   记录这次方法调用,如果日志允许缓存，则缓存方法
* cls is the method whose cache should be filled. 
方法缓存所在的类
* implementer is the class that owns the implementation in question.
 implementer是imp的拥有者的类
**********************************************************************/
static void
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (objcMsgLogEnabled) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    cache_fill (cls, sel, imp, receiver);
}
```

```objectivec
void cache_fill(Class cls, SEL sel, IMP imp, id receiver)
{
 // Define DEBUG_TASK_THREADS to debug crashes when task_threads() is failing.
 // 定义 DEBUG_TASK_THREADS 用于task_threads()失败时的调试，这里涉及到mach的task，暂不做更深入的分析
#if !DEBUG_TASK_THREADS
    // 会在加锁的情况下，对垃圾缓存进行回收
    // 给互斥锁上锁，因为对缓存的写和读的操作都很频繁，这里选择互斥锁
    mutex_locker_t lock(cacheUpdateLock);
    cache_fill_nolock(cls, sel, imp, receiver);
#else
    // 这里用作调试，检测是否有其他线程访问缓存,后面会有详细的分析
    _collecting_in_critical(); 
    return;
#endif
}
```

这里上锁的方式比较特殊，值得一说，因为这只看到了lock，问什么没有看到unlock？

先看一下mutex_locker_t的定义

```objectivec
class mutex_locker_t : nocopy_t {
    mutex_t& lock;
  public:
   // 构造方法 后面的: lock(newLock) 等价于 lock=newlock
    mutex_locker_t(mutex_t& newLock): lock(newLock) 
         {
            lock.lock();
    }
    ~mutex_locker_t() {
        lock.unlock();
    }
};
```

这里是C++语法，不使用new 在栈上创建一个对象，会在作用域结束时，自动销毁，并调用析构方法，更多介绍，可参考[C++ Object without new](https://stackoverflow.com/questions/1764831/c-object-without-new)

#### cache_fill_nolock

```objectivec
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{
    // 验证缓存更新锁已经上锁 互斥锁
    cacheUpdateLock.assertLocked();

    // Never cache before +initialize is done
    // 在class没有初始化完成时，不进行缓存
    if (!cls->isInitialized()) return;

    // Make sure the entry wasn't added to the cache by some other thread 
    // before we grabbed the cacheUpdateLock.
    // 确保在获得cacheUpdateLock之前，imp是否已经被其他线程加入了缓存中
    if (cache_getImp(cls, sel)) return;

    // 获得类的缓存
    cache_t *cache = &cls->cache;
    // typedef objc_sel * sel;
    // typedef uintptr_t cache_key_t;
    // 把全局的sel指针地址转成 指针地址 
    cache_key_t key = (cache_key_t)sel;

    // Use the cache as-is if it is less than 3/4 full
    // 正常使用，如果缓存不足容量的3/4
    // 占用的bucket个数
    mask_t newOccupied = cache->occupied() + 1; 
    // capacity=_mask ? _mask+1 : 0; 缓存容量，所有bucket的个数
    mask_t capacity = cache->capacity();
    
    // 如果 cache 如果占用的bucktets个数是0，且_bucktes是初始值，则进行内存分配
    if (cache->isConstantEmptyCache()) {
        // Cache is read-only. Replace it.
        // INIT_CACHE_SIZE=1<<2=4
        // 初始状态的缓存是只读的,因为默认的是分配在静态内存空间上的，重新分配可读写内存以替换掉它
        // 问题1
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
    }
    else if (newOccupied <= capacity / 4 * 3) {
        // Cache is less than 3/4 full. Use it as-is.
        // 如果占用的bucket数量不足3/4,正常使用
    }
    else {
        // Cache is too full. Expand it.
        // >3/4,容量倍增
        cache->expand();
    }

    // Scan for the first unused slot and insert there.  找第一个未使用的占位buckets
    // There is guaranteed to be an empty slot because the 
    // minimum size is 4 and we resized at 3/4 full. 
    // 保证会有一个空的buckets，因为初始化状态会有4个，在3/4满时，会倍增，
    bucket_t *bucket = cache->find(key, receiver);
    if (bucket->key() == 0) _occupied++; // bucket->key()!=0 可能是覆写，无需修改_ocupied;
    bucket->set(key, imp);
}
```

`find()` 部分已经在中[TODO]介绍过,这里稍微一提

```objectivec
bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0);
    bucket_t *b = buckets();  // 缓存hash表首地址
    mask_t m = mask();
    // 把 SEL 和 mask 进行与运算 ，作为优先索引，优化缓存查找算法的效率 
    mask_t begin = (mask_t)(key & mask);
    mask_t i = begin;
    do {
        // 如果是空的bucket 或着是同一个SEL，则返回
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
       // 如果已经是被占用的，则向后查找空的bucket
    } while ((i = (i+1) & mask != begin);

    // hack 出错的情况
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}
```

看一下`isConstantEmptyCache`是 判断的什么

```objectivec
/*
* 判断是不是初始状态
* 如果占用的bucktets是0，且_bucktes是初始值，在常量取分配的一块内存，不可写
*/
bool cache_t::isConstantEmptyCache()
{
    // 占用的buckets个数是0，且 _bucktes=&_objc_empty_cache
    return  occupied() == 0  &&  buckets() == emptyBucketsForCapacity(capacity(), false);
}

/*
 * 判断是否是初始状态，不可写的情况
 */
bucket_t *emptyBucketsForCapacity(mask_t capacity, bool allocate = true)
{
    // 验证缓存更新锁已经上锁
    cacheUpdateLock.assertLocked();

    // fixme put end marker inline when capacity+1 malloc is inefficient
    // size_t bytes =sizeof(bucket_t) * (cap + 1);
    size_t bytes = cache_t::bytesForCapacity(capacity);

    // Use _objc_empty_cache if the buckets is small enough.
    // 如果buckets 足够小，使用初始值
    if (bytes <= EMPTY_BYTES) {
        return (bucket_t *)&_objc_empty_cache;
    }

     // ... 其他部分代码，暂不分析
    return emptyBucketsList[index]; 
}
```

#### 使用汇编定义只读的结构体空间

对应上面的问题，`_objc_empty_cache`是怎么定义的？

```objectivec
/***********************************************************************
* Pointers used by compiled class objects
* These use asm to avoid conflicts with the compiler's internal declarations
**********************************************************************/
// EMPTY_BYTES includes space for a cache end marker bucket.
// This end marker doesn't actually have the wrap-around pointer 
// because cache scans always find an empty bucket before they might wrap.
// 1024 buckets is fairly common.
// EMPTY_BYTES包含一个缓存和一个标记bucket的空间，这个最后的结束标志实际上并不包含一个指针
// 因为缓存扫描总是找一个空的bucket,在他们填充之bucket之前，1024个buckets较为普遍
#if DEBUG
    // Use a smaller size to exercise heap-allocated empty caches.
#   define EMPTY_BYTES ((8+1)*16)
#else
#   define EMPTY_BYTES ((1024+1)*16)
#endif

#define stringize(x) #x
#define stringize2(x) stringize(x)

// 在代码段中，也有可能包含一些只读的常数变量，例如字符串常量等(摘自维基百科)
// "cache" is cache->buckets; "vtable" is cache->mask/occupied
// hack to avoid conflicts with compiler's internal declaration
asm("\n .section __TEXT,__const" 
    "\n .globl __objc_empty_vtable"
    "\n .set __objc_empty_vtable, 0"
    "\n .globl __objc_empty_cache"
    "\n .align 3"
    "\n __objc_empty_cache: .space " stringize2(EMPTY_BYTES)
    );
```

这里使用汇编定义`__objc_empty_cache`，是避免和内部的`__objc_empty_cache`声明有冲突

并定义在了代码段的`__TEXT,__const`区，如果不了解段（segment）和区（section）的区别。可参考[TODO]

>  `.section __TEXT,__const`The compiler places all data declared `const` and all jump tables it generates for switch statements in this section.
>
>  编译器把所有声明为const的数据放在这个区

更多信息，可参考 [OS X Assembler Reference](https://developer.apple.com/library/content/documentation/DeveloperTools/Reference/Assembler/040-Assembler_Directives/asm_directives.html)

> `.space size , fill`
>
> This directive emits `size` bytes, each of value `fill`. Both `size` and `fill` are absolute expressions. If the comma and `fill` are omitted, `fill` is assumed to be zero.
>
> 这个指令会把size大小的控件，每个都填充fill对应的值，size 和 fill 都可以是表达式，如果省去fill，默认是填充0

即这里的汇编指令的作用是，在代码段的`__TEXT,__const`区，分配了一个((1024+1)*16) 大小，全部填充0的区域。因为  16 = sizeof(bucket_t),所以分配了1024+1个bucket_t大小的空间，最后一个结束标记bucket。

如果是初始化的状态，则会重新分配可读写的内存，用于缓存

```objectivec
/*
 缓存容量倍增
 #if __LP64__
 typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
 #endif
 */
void cache_t::expand()
{
    cacheUpdateLock.assertLocked(); // 在内存扩张时，确认已加锁（互斥锁）
    
    // 老的缓存容量
    uint32_t oldCapacity = capacity();
    // 新的缓存容量是原来的两倍,默认是4
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    // 如果 溢出了，则不再扩张
    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask 
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);
}
```



```objectivec
/*
* 缓存容量的扩增和垃圾缓存回收
*/
void cache_t::reallocate(mask_t oldCapacity, mask_t newCapacity)
{  
    // 现在处于加锁的状态
    // 如果没有占用的bucket且是初始的状态，就无需释放
    bool freeOld = canBeFreed();

    // 废弃的缓存，将要变成垃圾
    bucket_t *oldBuckets = buckets();
    // 分配新容量的bucket_t，并设置好最后的标志结束buckets
    bucket_t *newBuckets = allocateBuckets(newCapacity);

    // Cache's old contents are not propagated. 
    // This is thought to save cache memory at the cost of extra cache fills.
    // fixme re-measure this
    // 缓存的旧内容不被保留
    // 这基于节约缓存的考虑，虽然会需要额外的缓存填充
    assert(newCapacity > 0);
    assert((uintptr_t)(mask_t)(newCapacity-1) == newCapacity-1);
    
    // 设置 cache->_buckets 和 cache->mask
    setBucketsAndMask(newBuckets, newCapacity - 1);
    
    if (freeOld) {
        // 为垃圾引用表分配空间，把缓存垃圾添加到垃圾引用表中
        cache_collect_free(oldBuckets, oldCapacity);
        // 满足垃圾回收的条件，则进行回收
        cache_collect(false);
    }
}

bool cache_t::canBeFreed()
{
    return !isConstantEmptyCache();
}
```



```objectivec
void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask)
{
    // objc_msgSend uses mask and buckets with no locks.
    // objc_msgSend会在不加锁的情况下使用mask和buckets
  
    // It is safe for objc_msgSend to see new buckets but old mask.
    // (It will get a cache miss but not overrun the buckets' bounds).
    // objc_msgSend使用新的bucktets，老的mask是安全的，
    //（它会找不到缓存，但不会buckets数组越界）
  
    // It is unsafe for objc_msgSend to see old buckets and new mask.
    // Therefore we write new buckets, wait a lot, then write new mask.
    // objc_msgSend使用旧的buckets和新的mask是不安全的
    // 因此我们写一个新的buckets，等一段时间，然后写一个新的mask
  
    // objc_msgSend reads mask first, then buckets.
    // objc_msgSend 读取缓存的时候  是先读mask，然后再是buckets
    
    // ensure other threads see buckets contents before buckets pointer
    // 确保其他线程先看到buckets内容
    mega_barrier();

    _buckets = newBuckets;
    
    // ensure other threads see new buckets before new mask
    // 确保其他线程在设置新mask之前，使用新的buckets，即避免旧的buckets和新的mask的组合
    // 问题1 使用内存屏障是怎么做到避免旧的buckets和新的mask的组合的呢？
    mega_barrier();
    
    _mask = newMask;
    
    _occupied = 0;
}
```

```c
//the actual code does use memory barriers when installing the new cache buckets

// __asm__(汇编语言:输出:输入:修饰词")
// __volatile__ 用于告诉编译器，严禁将此处的汇编语句与其它的语句重组合优化。即：原原本本按原来的样子处理
// CPUID指令是intel IA32架构下获得CPU信息的汇编指令，可以得到CPU类型，型号，厂商信息，商标信息，序列号，缓存等一系列CPU相关的东西。
// CPUID指令一般使用使用eax作为输入参数（某些时候会用到ecx），eax、ebx、ecx、edx作为输出参数。例如这样的汇编代码
// EAX=0：获取CPU的Vendor ID
//对于Intel的CPU,，返回的字符串永远是：GenuineIntel。对应在三个寄存器中的值如下：EBX=756E6547h，EDX=49656E69h，ECX=6C65746Eh
// "memory"：将不重新排序该段内嵌汇编指令与前面的指令，不使用寄存器作为缓存。
// "=a" (_clbr) : "0" (0) :代表输入输出都是用rax寄存器
// "rbx", "rcx", "rdx", "cc", "memory" :告诉编译器在这条指令里面我们会修改什么值，这里是rbx，rcx，rdx，cc寄存器和内存
// "cc" :状态寄存器又名条件码寄存器(Condition code register)
// 作用：阻止指令重排,并获得cpu信息
#define mega_barrier()
     unsigned long _clbr;
     __asm__ __volatile__("cpuid": "=a" (_clbr) : "0" (0) : "rbx", "rcx", "rdx", "cc", "memory" );
```

更多内嵌汇编，可参考[GCC内嵌汇编](https://dirtysalt.github.io/gcc-asm.html)。

问题1 ：使用内存屏障是怎么做到避免旧的buckets和新的mask的组合的呢？

大多数现代计算机为了提高性能而采取[乱序执行](https://zh.wikipedia.org/wiki/%E4%B9%B1%E5%BA%8F%E6%89%A7%E8%A1%8C)，乱序执行就可能导致产生我们不想要的结果，例如这里，如果没有` mega_barrier();`,在执行时，就可能导致`_mask = newMask;`先于` _buckets = newBuckets;`，导致objc_msgSend在无锁获取mask和_buckets时，出现旧的buckets和新的mask的组合，导致在缓存查找时出现越界。更多内存屏障和乱序执行的知识，可参考[TODO]。

#### 缓存垃圾回收过程

```objectivec
/***********************************************************************
* cache_collect_free.  
* Add the specified malloc'd memory to the list
* of them to free at some later point.
* 把指定的堆内存地址添加到垃圾引用表中，在以后的某一时刻释放（垃圾达到阈值且没有其他线程访问缓存时）。
* size is used for the collection threshold. It does not have to be 
* precisely the block's size.
* 大小用来和阈值进行比较，如果大于阈值则回收缓存，否则继续累积，没必要是精确的块大小
* Cache locks: cacheUpdateLock must be held by the caller.
**********************************************************************/
static void cache_collect_free(bucket_t *data, mask_t capacity)
{
    cacheUpdateLock.assertLocked();
   
    // 记录垃圾缓存，比较简单且无关，不做分析
    if (PrintCaches) recordDeadCache(capacity);

    //创建垃圾引用表，当垃圾足够大时，才清理
    _garbage_make_room ();
    // 记录所有垃圾缓存的大小
    garbage_byte_size += cache_t::bytesForCapacity(capacity);
    // garbage_count++  把垃圾缓存的引用添加到垃圾引用表中
    garbage_refs[garbage_count++] = data;
}
```

```objectivec
/***********************************************************************
* _garbage_make_room.  Ensure that there is enough room for at least
* one more ref in the garbage.
* 确保有足够的空间用于存放垃圾引用
**********************************************************************/

// amount of memory represented by all refs in the garbage
// 所有的垃圾缓存的大小
static size_t garbage_byte_size = 0;

// do not empty the garbage until garbage_byte_size gets at least this big
// 不清空垃圾，直到垃圾的大小到达一定的大小 阈值 = 32 *1024；
static size_t garbage_threshold = 32*1024;

// table of refs to free
// 需要释放的引用表
static bucket_t **garbage_refs = 0;

// current number of refs in garbage_refs
// 当前的垃圾引用数量
static size_t garbage_count = 0;

// capacity of current garbage_refs
// 当前的垃圾引用表的容量 初始值是 INIT_GARBAGE_COUNT=128
static size_t garbage_max = 0;

// capacity of initial garbage_refs
// 初始化状态的垃圾引用表的容量
enum {
    INIT_GARBAGE_COUNT = 128
};

// 创建垃圾引用表，当垃圾足够大时，才清理
static void _garbage_make_room(void)
{
    static int first = 1;

    // Create the collection table the first time it is needed
    // 首次时，创建垃圾表 二级指针 ：一级指针指向一次分配的缓存buckets（可能是2^2,2^3,2^4,...个buckets）
    // 二级指针指向单个bucket
    if (first)
    {
        first = 0;
        garbage_refs = (bucket_t**)
            malloc(INIT_GARBAGE_COUNT * sizeof(void *));
        garbage_max = INIT_GARBAGE_COUNT;
    }

    // Double the table if it is full 如果垃圾引用表满了，倍增
    else if (garbage_count == garbage_max)
    {
        garbage_refs = (bucket_t**)
            realloc(garbage_refs, garbage_max * 2 * sizeof(void *));
        garbage_max *= 2;
    }
}
```

```objectivec
/***********************************************************************
* cache_collect.  
* Try to free accumulated dead caches.
* 尝试去清理积累的垃圾缓存，如果达到阈值且没有其他线程访问缓存时，进行垃圾回收
* collectALot tries harder to free memory.
* collectALot 强制回收
* Cache locks: cacheUpdateLock must be held by the caller.
**********************************************************************/
void cache_collect(bool collectALot)
{
    cacheUpdateLock.assertLocked();

    // Done if the garbage is not full
    // 如果垃圾的大小还没有到达阈值，而且不要强制时，则暂时不回收
    if (garbage_byte_size < garbage_threshold  &&  !collectALot) {
        return;
    }

    // Synchronize collection with objc_msgSend and other cache readers
    // 回收垃圾时，和其他线程的缓存访问者，保持同步
    if (!collectALot) {
        // 因为其他线程在读取缓存时，是不加锁的，所以在当前情况下，可能有其他线程在读取缓存，并可能使用垃圾缓存
        if (_collecting_in_critical ()) {
            // objc_msgSend (or other cache reader) is currently looking in
            // the cache and might still be using some garbage.
            // 其他线程可能当前正在访问缓存，并使用垃圾缓存
            if (PrintCaches) {
                _objc_inform ("CACHES: not collecting; "
                              "objc_msgSend in progress");
            }
            return;
        }
    } 
    else {
        // No excuses. 强制回收时，只要有其他线程正在访问缓存，就不能垃圾回收，只能空转等待，直到回收完成
        // 只有flushCaches 中会调用cache_collect(true),即在类初始化时，添加分类方法时，等修改类结构时，才会强制垃圾回收
        while (_collecting_in_critical()) 
            ;
    }

    // No cache readers in progress - garbage is now deletable
    // 没有其他线程在访问缓存，可以清除缓存

    // Log our progress 记录清理过程
    if (PrintCaches) {
        cache_collections++; // 垃圾清理数量++
        _objc_inform ("CACHES: COLLECTING %zu bytes (%zu allocations, %zu collections)", garbage_byte_size, cache_allocations, cache_collections);
    }
    
    // Dispose all refs now in the garbage
    // 释放所有的垃圾应用表中的缓存垃圾
    // Erase each entry so debugging tools don't see stale pointers.
    // 擦除所有的入口(指针置为nil)，这样调试工具，就不会得到野指针
    while (garbage_count--) {
        auto dead = garbage_refs[garbage_count];
        garbage_refs[garbage_count] = nil;
        free(dead);
    }
    
    // Clear the garbage count and total size indicator
    // 垃圾数量 和垃圾大小置为0
    garbage_count = 0;
    garbage_byte_size = 0;

    //... 日志输出
}

```

这里，总结一下垃圾回收的过程，在每次缓存扩容时 （collectALot=false;）

- 现将先前的缓存，作为垃圾缓存添加到垃圾引用表中；
- 如果垃圾缓存的大小，没有达到阈值，则不仅回收；
- 如果超过阈值，便检测当前是否有其他线程访问缓存，如果有，则直接return,等下一次再删；

如果collectALot=true;

- 无论缓存总大小，如果当前有其他线程访问缓存，则一直空转，直到没有其他线程访问时，清空缓存。即强制清空缓存。





#### 检测其他线程是否在执行某块代码

`_collecting_in_critical()`，即检测如果当前有其他线程方法缓存，则不进行垃圾回收，如果没有，则进行垃圾回收。更多信息请移步[Concurrent Memory Deallocation in the Objective-C Runtime](https://www.mikeash.com/pyblog/friday-qa-2015-05-29-concurrent-memory-deallocation-in-the-objective-c-runtime.html)

```objectivec
/***********************************************************************
* _collecting_in_critical.
* Returns TRUE if some thread is currently executing a cache-reading 
* function. Collection of cache garbage is not allowed when a cache-
* reading function is in progress because it might still be using 
* the garbage memory.
 返回TRUE，如果一些线程正在执行缓存读取函数。 当有一个缓存读取的函数正在执行时，缓存垃圾的释放是不允许的，因为，在同时另一个线程，可能依然使用者在使用那块弃用的缓存垃圾，即保证线程安全
**********************************************************************/
// 关键的PC位置存储在全局变量中,这两个全局变量定义在objc-msg-x86_64.s中
// 这也是一个汇编方法在开始和结束有一个ENTRY 和一个END_ENTRY的作用
// 获得rip寄存器的值，便可知道当前线程是否在执行某个方法
//  objc_entryPoints[region]<= pc <= objc_exitPoints[region]
OBJC_EXPORT uintptr_t objc_entryPoints[];
OBJC_EXPORT uintptr_t objc_exitPoints[];
/**
 函数本身不需要参数，返回值是int 类型的，使用起来就像一个bool标志位一样，用来标识在一个关键函数中是否有多个线程
 它在释放残留的内存垃圾之前调用。runtime实际上有两种独立的模式：一种是留下垃圾直到下次再有其他线程进入临界函数。另一个是不断的循环直到清除干净，而且通常会同时释放这些垃圾内存。
 由于缓存释放需要检查进程中的每个线程的状态，因此它是相对低效的。但是如果objc_msgSend只用考虑单线程的环境下，它的执行效率将会非常高。这值得做出权衡。
 @return int
 */
static int _collecting_in_critical(void)
{
#if TARGET_OS_WIN32
    return TRUE;
#else
    thread_act_port_array_t threads;
    unsigned number;
    unsigned count;
    kern_return_t ret;
    int result;

    // 当前线程
    mach_port_t mythread = pthread_mach_thread_np(pthread_self());

    // Get a list of all the threads in the current task
#if !DEBUG_TASK_THREADS
    // 获得线程信息的API位于mach层面。task_threads 获得了给定任务中所有线程的线程列表，并且这些代码使用它来获得其所在进程中的其他线程
    // 它返回了一组包含了多个thread_t值的threads数组，并且可以获得数组元素的个数，然后它会遍历这些元素
    ret = task_threads(mach_task_self(), &threads, &number);
#else
    ret = objc_task_threads(mach_task_self(), &threads, &number);
#endif

    if (ret != KERN_SUCCESS) {
        // See DEBUG_TASK_THREADS below to help debug this.
        _objc_fatal("task_threads failed (result 0x%x)\n", ret);
    }

    // Check whether any thread is in the cache lookup code
    // 检测是否有其他线程在访问缓存
    result = FALSE;
    for (count = 0; count < number; count++)
    {
        int region;
        uintptr_t pc;

        // Don't bother checking ourselves 如果是当前线程就不用检查了
        if (threads[count] == mythread)
            continue;

        // Find out where thread is executing 获得对应线程rip寄存器值
        pc = _get_pc_for_thread (threads[count]);

        // Check for bad status, and if so, assume the worse (can't collect) 
        // 如果获得pc失败，则不进行垃圾回收
        if (pc == PC_SENTINEL)
        {
            result = TRUE;
            goto done;
        }
        
        // Check whether it is in the cache lookup code 检测是否在执行cache lookup
        // objc_entryPoints 和 objc_exitPoints 都以0结尾
        for (region = 0; objc_entryPoints[region] != 0; region++)
        {
            if ((pc >= objc_entryPoints[region]) &&
                (pc <= objc_exitPoints[region])) 
            {
                result = TRUE;
                goto done;
            }
        }
    }

 done:
    // Deallocate the port rights for the threads
    for (count = 0; count < number; count++) {
        mach_port_deallocate(mach_task_self (), threads[count]);
    }

    // Deallocate the thread list
    vm_deallocate (mach_task_self (), (vm_address_t) threads, sizeof(threads[0]) * number);

    // Return our finding
    return result;
#endif
}
```

```objectivec
// A sentinel (magic value) to report bad thread_get_state status.
// 用一个哨兵（魔数）去报告损坏的线程状态码
// Must not be a valid PC. 
// 一定不是一个有效的当前指令地址
// Must not be zero - thread_get_state() on a new thread returns PC == 0.
#define PC_SENTINEL  1

static uintptr_t _get_pc_for_thread(thread_t thread)
#if defined(__x86_64__)
{
    x86_thread_state64_t			state;
    unsigned int count = x86_THREAD_STATE64_COUNT;
    // 获得目标线程的寄存器状态。它位于一个独立的函数中的主要原因是寄存器状态的结构是特定于系统架构的，因为每个架构下有着不同的寄存器。这就意味着这个函数对于每种支持的架构需要一个独立的实现，尽管每种实现都几乎是一样的。这里有一个关于x86-64架构下的实现
    kern_return_t okay = thread_get_state (thread, x86_THREAD_STATE64, (thread_state_t)&state, &count);
    // 获得对应线程rip寄存器值 (Register Instruction Pointer)
    return (okay == KERN_SUCCESS) ? state.__rip : PC_SENTINEL;
}
// ... 其他
}
#endif
```