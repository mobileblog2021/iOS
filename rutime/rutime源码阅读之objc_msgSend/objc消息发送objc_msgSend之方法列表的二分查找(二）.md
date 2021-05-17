

# 详细分析objc_msgSend

> 文章中的代码都是在 `x86_64` 架构下运行的，arm架构的代码大同小异。
>
> 这里分析的版本是objc4-706

## 写在前面

上一篇文章分析了`objc_msgSend`的汇编部分，本篇将继续向下分析。详细代码参见于[coding](https://git.coding.net/terrylxt/SuningEBuy.git)

## 链接上一篇文章

当`objc_msgSend`没有在缓存中查找到对应的方法后，便去class继承链查找

```objectivec
/***********************************************************************
* _class_lookupMethodAndLoadCache.
* Method lookup for dispatchers ONLY. OTHER CODE SHOULD USE lookUpImp().
* This lookup avoids optimistic cache scan because the dispatcher 
* already tried that.
* 这个方法只会被派发器（objc_msgSend的汇编部分）调用,其他代码应该使用lookUpImp()
* 例如 lookUpImpOrForward， lookUpImpOrNil
* 这个查询方法避免了在缓存中查找的过程，因为objc_msgSend前面部分，已经使用汇编代码
* 查过了。即cache=NO
**********************************************************************/
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```

```objectivec
/***********************************************************************
* lookUpImpOrForward.
* The standard IMP lookup. 
* initialize==NO tries to avoid +initialize (but sometimes fails)
* cache==NO skips optimistic unlocked lookup (but uses cache elsewhere)
* Most callers should use initialize==YES and cache==YES.
* inst is an instance of cls or a subclass thereof, or nil if none is known. 
*   If cls is an un-initialized metaclass then a non-nil inst is faster.
* May return _objc_msgForward_impcache. IMPs destined for external use 
*   must be converted to _objc_msgForward or _objc_msgForward_stret.
*   If you don't want forwarding at all, use lookUpImpOrNil() instead.
 
* initialize==NO 尝试去避免+initialize调用，但有时会失效
* cache==NO 跳过乐观的缓存查找，因为之前已经查找过过了，(但其他地方使用缓存查找)
* 大多数调用者需要设置initialize==YES and cache==YES.
* inst 是cls的实例,或者子类的实例，或者nil，如果不知的话
* 如果cls 是一个未初始化的元类，那个一个非nil的inst会更快.
* 可能会返回_objc_msgForward_impcache的结果，IMP，一定是被_objc_msgForward或着_objc_msgForward_stret使用的
* 如果不想转发，使用lookUpImpOrNil()替代
**********************************************************************/
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    Method meth;
    bool triedResolver = NO;

    // 声明 rwlock_t runtimeLock
    // runtimeLock是一个读写锁,因为修改类结构的操作数量远小于读取操作的数量，所以使用读写锁
    runtimeLock.assertUnlocked();

    // Optimistic cache lookup 乐观的缓存查找
    // objc_msgSend在汇编部分已经使用CacheLookup查过了，没有查找才执行到这里
    // 所以这里cache=NO，无需再次查询
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    // objc在初始化时（_objc_init），在映射镜像时（map_2_images），会对class进行初始化(realizeClass)
    // 这里是确保消息发送前，class已经初始化完成
    // 这部分会在下一个系列文章，类的加载过程中详细分析
    if (!cls->isRealized()) {
        rwlock_writer_t lock(runtimeLock);
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        // 如果没有调用过+initialize，且initialize=YES 则调用+initialize
        _class_initialize (_class_getNonMetaClass(cls, inst));
    }

    // The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    // be added but ignored indefinitely because the cache was re-filled 
    // with the old value after the cache flush on behalf of the category.
    // 这个锁保证method-lookup 和 cache-fill的原子性,因为同时，可能有方法的添加;
    // 如果不加锁，一个已经加载的分类可能被忽略。
    // 因为在分类对应的方法缓存被更新之后，缓存依旧是旧值（这里需要调整）。
 retry:
    runtimeLock.read(); // 查找方法，所以设置读锁

    // Try this class's cache.
    // 这里和上面乐观的缓存查找cache_getImp的区别是，这里是加锁状态，保证在没有写操作的状态下，在缓存中查找
    // cache_getImp 也是通过汇编实现，直接调用上一篇介绍的CacheLookup，只不过这里不是调用IMP，而是返回
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.
    // 在当前类即分类中查找方法（暂不往继承链上找）
    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        // 如果找到了，则将方法的实现加入缓存中
        log_and_fill_cache(cls, meth->imp, sel, inst, cls);
        imp = meth->imp;
        goto done;
    }

    // ... 没找到 去父类的缓存和方法列表中查找，在后面继续分析
 done:
    runtimeLock.unlockRead();

    return imp;
}
```

如果在当前类的缓存中没有找到，则会先遍历当前类的方法列表。我们看是怎么查找的

```objectivec
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    // 确认已经加过锁了，对锁的管理，见后面仔细分析
    runtimeLock.assertLocked();

    // 确认已经初始化过了,只有在初始化之后，cls->data()才是可读写数据class_rw_t
    assert(cls->isRealized());
    // fixme nil cls? 
    // fixme nil sel?
    
    // 存储类的方法、属性和遵循的协议等信息的地方
    // #define FAST_DATA_MASK          0x00007ffffffffff8UL
    // (class_rw_t *)(cls->bits & FAST_DATA_MASK)
    // 因为 class_rw_t * 指针存于第 [3, 47] 位
    // 这里少稍微一提，更多信息见[TODO]
    class_rw_t *data=cls->data();
    // methods是class所有的方法包括类和分类的方法，是一个二维数组
    // 通过lldb 输出，可以验证一维对应一个镜像(image:例如TestMaclibxpc.dylib，AppKit等)
    // 二维代表当前镜像下类和分类的所有方法
    // 以后还有以源码为基础的分析[TODO]
    method_array_t methods=data->methods;
    
    // 对二维数组进行遍历
    method_list_t **mlists=methods.beginLists();
    method_list_t **end = methods.endLists();
    for (;mlists != end;++mlists)
    {
        method_t *m = search_method_list(*mlists, sel);  // 添加断点处，调试结果见下方
        if (m) return m;
    }

    return nil;
}
```

 关于 method_array_t<method_t, method_list_t>  继承自template <typename Element, typename List> list_array_tt ;

list_array_tt 泛型实现用于对多种类型的元数据进行存储，是一个二维数组

其中 Element 表示元数据的类型，比如 method_t，而 List 则表示用于存储元数据的一维数组，比如 method_list_t。

 list_array_tt 有三种状态:

 自身为空，可以类比为 [[]]
 它只有一个指针，指向一个元数据的集合，可以类比为 [[1, 2]]
 它有多个指针，指向多个元数据的集合，可以类比为 [[1, 2], [3, 4]]

```objectivec
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

1. `name` 表示的是方法的名称，用于唯一标识某个方法，比如 `@selector(viewWillAppear:)` ；
2. `types` 表示的是方法的返回值和参数类型（详细信息可以查阅苹果官方文档中的 [Type Encodings](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1) ）；
3. `imp` 是一个函数指针，指向方法的实现；
4. `SortBySELAddress` 顾名思义，是一个根据 `name` 的地址对方法进行排序的内部结构体。

```objectivec
static void 
fixupMethodList(method_list_t *mlist, bool bundleCopy, bool sort)
{
    // ... 其他
    // Sort by selector address.
    if (sort) {
        method_t::SortBySELAddress sorter;
        std::stable_sort(mlist->begin(), mlist->end(), sorter);
    }
    // ... 其他
}

```

```shell
 # 执行下一个循环后调试，查找二维数组中的值即 method_list_t *$4=methods.get(1)
 (lldb) p *mlists
 (method_list_t *) $0 = 0x00000001000031d0
 (lldb) p $0->get(0)
 (method_t) $1 = {
   name = "hello"
   types = 0x0000000100002b55 "v16@0:8"
   imp = 0x0000000100001be0 (TestMac`-[LTObject hello] at LTObject.m:15)
 }
 (lldb) p $0->get(1)
 (method_t) $2 = {
   name = "world"
   types = 0x0000000100002b5d "@16@0:8"
   imp = 0x0000000100001c10 (TestMac`-[LTObject world] at LTObject.h:17)
 }
 (lldb) p $0->get(2)
 (method_t) $3 = {
   name = "setWorld:"
   types = 0x0000000100002b65 "v24@0:8@16"
   imp = 0x0000000100001c30 (TestMac`-[LTObject setWorld:] at LTObject.h:17)
 }
 (lldb) p $0->get(3)
 (method_t) $4 = {
   name = "hello_category"
   types = 0x0000000100002b55 "v16@0:8"
   imp = 0x0000000100001cb0 (TestMac`-[LTObject(Category) hello_category] at LTObject.m:23)
 }
 (lldb) p $0->get(4)
 (method_t) $5 = {
   name = ".cxx_destruct"
   types = 0x0000000100002b55 "v16@0:8"
   imp = 0x0000000100001c70 (TestMac`-[LTObject .cxx_destruct] at LTObject.m:13)
 }
 (lldb) p $0->get(5)
 Assertion failed: (i < count), function get, file /Users/terry/WorkSpace/ios/SuningEBuy/objc4_706/objc-runtime-master_706/runtime/objc-runtime-new.h, line 119.
 error: Execution was interrupted, reason: signal SIGABRT.
 The process has been returned to the state before expression evaluation.
```

```shell
# 执行下一个循环后调试，查找二维数组中的值即 method_list_t *$4=methods.get(1)
(lldb) p *mlists
(method_list_t *) $4 = 0x00007fffb5d95260
(lldb) p $4->get(0)
(method_t) $5 = {
  name = "load"
  types = 0x00007fffad0e3b68 "v16@0:8"
  imp = 0x00007fffad0dc879 (libxpc.dylib`+[OS_xpc_mach_send load]) # libxpc.dylib
}

# 执行下一个循环后调试，查找二维数组中的值即 method_list_t *$6=methods.get(2)
(lldb) p *mlists
(method_list_t *) $6 = 0x00007fffb1f94578
(lldb) p $6->get(0)
(method_t) $7 = {
  name = "_kitNewObjectSetVersion:"
  types = 0x00007fff95f27b92 "v24@0:8q16"
  imp = 0x00007fff951ce9dc (AppKit`+[NSObject(NSSetVersionHacks) _kitNewObjectSetVersion:]) #AppKit
}
```

下面我们来看一下search_method_list()的实现

```objectivec
/***********************************************************************
* getMethodNoSuper_nolock
* fixme
* Locking: runtimeLock must be read- or write-locked by the caller
**********************************************************************/
static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    // 属于类的加载过程，暂不做分析 [TODO]
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    // 如果已经对方法进行过排序，则使用二分法查找，否则使用线性查找
    // 关于编译器内置方法__builtin_expect(),请移步[TODO]
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // Linear search of unsorted method list
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }
    // ... 调试代码
    return nil;
}
```

进行二分查找，数组必须是有序的，我们来看一下,class中的方法列表是否按照SEL指针地址有序排列？

这里有一个基础：即SEL 指针地址是全局的，只和名字有关系，到底是加号方法，还是减号方法，在哪个类中，都没有关系。详细了解请移步[TODO]

```shell
 (lldb) p @selector(hello)
 (SEL) $6 = "hello"
 (lldb) p/x @selector(hello) # 对应上方lldb输出 $0->get(0) 
 (SEL) $7 = 0x0000000100002470 "hello"
 (lldb) p/x @selector(world) # 对应上方lldb输出 $0->get(1) 
 (SEL) $8 = 0x0000000100002484 "world"
 (lldb) p/x @selector(setWorld:) # 对应上方lldb输出 $0->get(2) 
 (SEL) $9 = 0x000000010000248a "setWorld:"
 (lldb) p/x @selector(hello_category) # 对应上方lldb输出 $0->get(3) 
 (SEL) $10 = 0x000000010000249b "hello_category"
 (lldb) p/x @selector(.cxx_destruct:) # 对应上方lldb输出 $0->get(4) 
 error: expected identifier
 error: expected expression
```

我们可以看到sel指针地址是递增的，这里有个例外，就是析构方法，没有对应的指针地址，当然这个方法也不允许我们直接调用。

好了，既然是有序的，当然就可以使用二分查找了。

```objectivec
static method_t *findMethodInSortedMethodList(SEL key, const method_list_t *list)
{
    assert(list);

    const method_t * const first = &list->first;
    const method_t *base = first;// 二分前的第一个位置
    const method_t *probe; // 探针 定位二分的位置
    uintptr_t keyValue = (uintptr_t)key;
    uint32_t count;
    
    // 二分法查找
    for (count = list->count; count != 0; count >>= 1) {
        // 获得中间的method_t地址
        probe = base + (count >> 1);
        
        uintptr_t probeValue = (uintptr_t)probe->name;
        
        // 找到了
        if (keyValue == probeValue) {
            // `probe` is a match.
            // Rewind looking for the *first* occurrence of this value.
            // This is required for correct category overrides.
            // 往前遍历 找第一次出现的值，处理在分类中覆盖方法的情况
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {
                probe--;
            }
            return (method_t *)probe;
        }
        
        // 要找的在后面 base 从后半部分开始 继续二分查找
        if (keyValue > probeValue) {
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}
```

#### 二分查找算法

上面的二分查找，稍微难理解一些,这里介绍两种二分搜索算法，省去与算法无关的部分

```objectivec
// 思路：在二分查找，通过改变起始和结束索引来缩小查找范围
int binarySearch(int searchArray[],int count,int key){
    int low=0;
    int high=count-1;
    while (low<=high) {
        int mid=(low+high)>>1;
        int midVal=searchArray[mid];
        if (midVal<key) {
            // key 在后边部分
            low=mid+1;
        }else if(midVal<key){
            // key 在前半部分
            high=low-1;
        }else{
            return mid;
        }
    }
    
    return -1;
}

// 思路：在二分查找，通过改变起始索引和数组长度来缩小查找范围
int binarySearch2(int searchArray[],int count,int key){
    
    int probeIndex;
    int baseIndex=0;
    
    for (; count!=0; count>>=1) {
        probeIndex=baseIndex+(count>>1);
        int probeVal=searchArray[probeIndex];
        if (probeVal==key) {
            return probeIndex;
        }
        
        if (probeVal<key) {
            baseIndex=probeIndex+1;
            count--;
        }
    }
    
    return -1;
    
}

int main(int argc, const char * argv[]) {
    int i, val, ret;
    int a[]={13,8,5,3,1};
    for(i=0; i<sizeof(a)/sizeof(int); i++)
        printf("%d\t", a[i]);
    while (1) {
        printf("\n请输人所要查找的元素：\n");
        scanf("%d",&val);
        
        ret = binarySearchDown(a,sizeof(a)/sizeof(int),val);
        if(-1 == ret)
            printf("查找失败 \n");
        else
            printf ("查找成功 %d \n",a[ret]);
       
    }
    return 0;
}
```
经比较，两者空间复杂度和时间复杂度（即log2 N）一样，可任选一种使用。

#### 扩展的二分查找

给一个降序的数列，比如 13，8，5，3，1   然后再给你一个数，比如2，然后你在数列里找到比2大的最小的数3，如果没有返回-1

```c
// 降序数列，查找被key大的最小的数
int binarySearchDown(int searchArray[],int count,int key){
    
    int probeIndex=-1;
    int baseIndex=0;
    int totoalCount=count;
    
    for (; count!=0; count>>=1) {
        probeIndex=baseIndex+(count>>1);
        int probeVal=searchArray[probeIndex];
        if (probeVal==key) {
            probeIndex--;
            // 谨慎的使用goto语句
            goto done;
        }
        
        if (probeVal>key) {
            baseIndex=probeIndex+1;
        }
    }
    // 对数组长度奇偶不同处理
    if (searchArray[probeIndex]<key) {
        probeIndex--;
    }

done:
    // 边界值
    if (probeIndex>=totoalCount) {
        return totoalCount-1;
    }
    
    return probeIndex;
    
}

// 升序数列，查找被key大的最小的数
int binarySearchUp(int searchArray[],int count,int key){
    
    int probeIndex=-1;
    int baseIndex=0;
    int totoalCount=count;
    
    for (; count!=0; count>>=1) {
        probeIndex=baseIndex+(count>>1);
        int probeVal=searchArray[probeIndex];
        if (probeVal==key) {
            probeIndex++;
            break;
        }
        
        if (probeVal<key) {
            baseIndex=probeIndex+1;
        }
    }
    
    if (searchArray[probeIndex]<key) {
        probeIndex++;
    }
    
    if (probeIndex>=totoalCount) {
        return -1;
    }
    return probeIndex;
    
}
```

#### lookUpImp*()系列方法

这里有两个比较相近的方法 `lookUpImpOrNil`和`lookUpImpOrForward`

```objectivec
/***********************************************************************
* lookUpImpOrNil.
* Like lookUpImpOrForward, but returns nil instead of _objc_msgForward_impcache
**********************************************************************/
IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
```

即两者的区别是，如果没有找到IMP，是否进行消息转发。

`_objc_msgForward_impcache`的更多分析，会在消息转发篇仔细分析。



#### 在继承链上查找

```objectivec
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
   // ... 前面其他

 retry:
    runtimeLock.read(); // 查找方法，所以设置读锁

    // Try this class's cache.
    // 这里和上面乐观的缓存查找cache_getImp的区别：这里是加锁状态，保证在没有写操作的状态下，在缓存中查找
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

    // Try superclass caches and method lists.
    // 缓存和当前类中都没有找到，尝试去父类的缓存和方法列表中查找
    curClass = cls;
    // 直到变量的NSObject，其父类为nil
    while ((curClass = curClass->superclass)) {
        // Superclass cache. 查找父类的缓存
        imp = cache_getImp(curClass, sel);
        if (imp) {
            if (imp != (IMP)_objc_msgForward_impcache) {
                // 如果在父类缓存中找到，且不是消息转发，则将该方法添加到当前类的缓存中，
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, imp, sel, inst, curClass);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                // 如果是消息转发，则不保存
                break;
            }
        }

        // Superclass method list.
        // 在缓存中没有找到，去方法列表中查找，同上
        meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
            imp = meth->imp;
            goto done;
        }
    }

   // ... 后面其他

 done:
    runtimeLock.unlockRead();

    return imp;
}
```

### 总结

这篇文章主要介绍当缓存中没有时，在当前类的方法列表中二分查找的过程。