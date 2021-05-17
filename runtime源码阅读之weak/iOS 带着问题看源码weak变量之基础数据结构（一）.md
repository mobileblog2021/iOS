# iOS 带着问题看源码weak变量之基础数据结构（一）

❓❓❓：weak变量是怎么存储的？一个简单的Hash表吗？


关于weak变量的存储相关的数据结构：
|数据结构 | 作用 | 内容 | 特点 |
|---|---|---|---|
|NSObject * | 对象地址 |
|StripedMap | 存储64个SideTable | key：对象地址  value：SideTable |多并发，提高性能 |
|SideTable | 两个功能存储，线程安全|存储对象的引用计数和弱引用表  | 分离锁，线程安全|
|weak_table_t | N个对象的弱引用表 | key：对象地址 value: weak_entry_t | |
|weak_entry_t | 1个对象的N个弱引用 | key：对象地址 value: 弱引用 | |
|weak_referrer_t | 1个弱引用 | |

对象的弱引用 比喻成 学生拥有的书

|数据结构 | 类比 |
|---|---|
|NSObject * | 1个学生 |
|StripedMap | 1个教学楼，N个教室 |
|SideTable | 1个教室：包括一把锁（分离锁）|
|weak_table_t | N个学生（同一个教室） |
|weak_entry_t | 1个学生的N本书 |
|weak_referrer_t | 1本书 | |


![图片](https://github.com/mobileblog2021/iOS/blob/main/runtime%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E4%B9%8Bweak/img/bj-b87a81af3a9ae7842d8ed1efb912c6a4d13fb11b.png?raw=true)

**这里看到的都是数组形态，但都是将对象的地址hash成index, 来取值，所以每一个都是一个hash表。**

我们通过源码看下这些数据结构：
```objective-c
// NSObject.mm
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;

static StripedMap<SideTable>& SideTables() {
    StripedMap<SideTable>& result = SideTablesMap.get();
    return result;
}

// 通过对象指针获取SideTable
objc_object *newObj = nil;
SideTable *newTable = &SideTables()[obj];

```

SideTablesMap的初始化
![图片](https://github.com/mobileblog2021/iOS/blob/main/runtime%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E4%B9%8Bweak/img/bj-5f7202fa360c42b0743ddffb0b796f353dfd9699.png?raw=true)
```objective-c
// NSObject.mm
// runtime初始化时（_objc_init），会调用这里
void arr_init(void) 
{
    AutoreleasePoolPage::init();
    SideTablesMap.init();
    _objc_associations_init();
}
```
```objective-c
// DenseMapExtras.h 
// We cannot use a C++ static initializer to initialize certain globals because
// libc calls us before our C++ initializers run. We also don't want a global
// pointer to some globals because of the extra indirection.
//
// ExplicitInit / LazyInit wrap doing it the hard way.
// 不能使用 C++ 静态初始化来初始化一些全局变量，因为libc调用这些全局变量在C++初始化之前，也不想增加额外的间接引用。
// ExplicitInit的作用是生成一个模板类型 Type 的实例。
template <typename Type>
class ExplicitInit {
    // 成员变量_storage是一个 sizeof(Type) 长度的 uint8_t 数组，而 uint8_t 占用一个字节，所以实际上_storage的长度跟一个 Type 实例所占用的内存是一样多的。
    alignas(Type) uint8_t _storage[sizeof(Type)];

public:
    template <typename... Ts>
    // 初始化生成的 Type 实例 赋值给_storage
    void init(Ts &&... Args) {
        new (_storage) Type(std::forward<Ts>(Args)...);
    }
    // 将 _storage 数组指针用reinterpret_cast<Type *>强转成了 Type * 类型指针，前面再加一个 *，说明返回的实际上是 Type 实例内存的首地址。
    Type &get() {
        return *reinterpret_cast<Type *>(_storage);
    }
};
```
所以`static StripedMap<SideTable>& SideTables()`函数实际上返回了一个全局静态`StripedMap<SideTable>`的实例。 下面是class StripedMap的部分定义
```objective-c
// StripedMap<T> is a map of void* -> T, sized appropriately 
// for cache-friendly lock striping. 
// For example, this may be used as StripedMap<spinlock_t>
// or as StripedMap<SomeStruct> where SomeStruct stores a spin lock.
// 
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };
    // 长度为 64，每个元素占 64 字节的数组
    // 即大小是 4096 = 64 * 64
    PaddedT array[StripeCount];

    // 据指针来计算哈希值，确定对应于array里面第几个元素。
    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        int index = ((addr >> 4) ^ (addr >> 9)) % StripeCount;
        return index;
    }

 public:
    // 重载了[]符号
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
    // 略其他方法
};
```
前面之道可以根据对象地址获取SideTable 即`SideTable *newTable = &SideTables()[obj];`我们再来看下SideTable的定义
```objective-c
struct SideTable {
    spinlock_t slock; // 自旋锁，适用于对锁的占用时间不长的场景，性能最好
    RefcountMap refcnts; // 当引用计数过大，isa不够存储，则存储在SideTable的该位置
    weak_table_t weak_table; // 全局弱引用表
   // 省略其他
};

/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 * 全局弱引用表，ids作为key, weak_entry_t structs作为value
 */
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};

struct weak_entry_t {
    // 被引用对象的地址
    DisguisedPtr<objc_object> referent; 
    union {
        struct {
            // 存放该对象所有弱引用的hash表，本质是个可变数组
            weak_referrer_t *referrers; 
            // 表示是否超过inlin_referrers大小，超过则使用referrers存储
            uintptr_t        out_of_line_ness : 2;
            // 当前数组的大小
            uintptr_t        num_refs : PTR_MINUS_2;
            // 数组长度-1，用于hash函数
            uintptr_t        mask;
            // hash 查找次数
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            // 只有4个元素的数组，默认情况下用它来存储弱引用的指针，当大于4个时，用referrers存储
            // #define WEAK_INLINE_COUNT 4
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
    // 略其他
};

// The address of a __weak variable.
// These pointers are stored disguised so memory analysis tools
// don't see lots of interior pointers from the weak table into objects.
typedef DisguisedPtr<objc_object *> weak_referrer_t;
```
