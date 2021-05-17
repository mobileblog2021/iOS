#### Block是什么？

block是带有自动变量（局部变量）的匿名函数,即Block = 函数 + 数据

```c++
// 代码摘自libclosure-67/Block_private.h

// Values for Block_layout->flags to describe block objects  
// Block_layout->flags 对应的值
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime // block正在释放(0位)
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime // 引用计数掩码 ([1-16]位)
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime // 需要释放
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler //  有copy 和 dispose 函数指针即Block_descriptor_2
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler  // 是否是全局的
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler // 有签名 即包含结构体Block_descriptor_3
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler // 包含扩展的布局
};

#define BLOCK_DESCRIPTOR_1 1
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size; // Block 大小
};

// 如果 Block_layout->flags & BLOCK_HAS_COPY_DISPOSE == 1 则Block_layout包含Block_descriptor_2结构体
#define BLOCK_DESCRIPTOR_2 1
struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};

// 如果 Block_layout->flags & BLOCK_HAS_SIGNATURE == 1 则Block_layout包含Block_descriptor_3结构体
#define BLOCK_DESCRIPTOR_3 1
struct Block_descriptor_3 {
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;  // Block_layout->invoke 函数签名
    const char *layout;     // contents depend on BLOCK_HAS_EXTENDED_LAYOUT
};

struct Block_layout {
    void *isa;   // block 类型
    volatile int32_t flags; // contains ref count
    int32_t reserved; // 保留字段
    void (*invoke)(void *, ...); // block调用的函数指针
    struct Block_descriptor_1 *descriptor;
    // requires BLOCK_HAS_COPY_DISPOSE
    // struct Block_descriptor_2 *descriptor_2;
    // requires BLOCK_HAS_SIGNATURE
    // struct Block_descriptor_3 *descriptor_3;
    // imported variables
};
```

#### Block的isa

block包含isa(指向 `_NSConcreteStackBlock` 或 `_NSConcreteGlobalBlock`)，这也是为什么Block能当做id类型的参数进行传递的原因。

> The isa field is set to the address of the external _NSConcreteStackBlock, which is a block of uninitialized memory supplied in libSystem, or _NSConcreteGlobalBlock if this is a static or file level Block literal.
> [摘自Clang文档](https://clang.llvm.org/docs/Block-ABI-Apple.html)

在iOS中，block isa常见的就是`_NSConcreteStackBlock`，`_NSConcreteMallocBlock`，`_NSConcreteGlobalBlock`

```c
// 代码摘自libclosure-67/Block.h

// Used by the compiler. Do not use these variables yourself.
BLOCK_EXPORT void * _NSConcreteGlobalBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteStackBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);

// 代码摘自libclosure-67/Block_private.h
// the raw data space for runtime classes for blocks
// class+meta used for stack, malloc, and collectable based blocks
BLOCK_EXPORT void * _NSConcreteMallocBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
```

**区分：** `__NSMallocBlock` `__NSMallocBlock__` `NSBlock` 

```c
__block int captureValue = 3;
void (^testBlock)() = ^{
    captureValue = 4;
};

Class blockClass = [testBlock class]; // __NSMallocBlock__
Class superClass = class_getSuperClass(blockClass) // __NSMallocBlock
Class topClass = class_getSuperClass(superClass)   // NSBlock

```

**区分：**`_NSConcreteStackBlock` `__NSStackBlock`

```assembly
# CoreFoundation and Foundation Framework

___CFMakeNSBlockClasses:

0000000000008029        leaq    0x4522b0(%rip), %rdi    ## literal pool for: “__NSStackBlock" /
# rdi 寄存器存储第一个参数
0000000000008030        callq   0x1d4858                ## symbol stub for: _objc_lookUpClass
# 调用_objc_lookUpClass("__NSStackBlock") 返回值（即__NSStackBlock）放在%rax寄存器
0000000000008035        movq    0x46f07c(%rip), %rdx    ## literal pool symbol address: __NSConcreteStackBlock  # %rdx存储第三个参数 （即__NSConcreteStackBlock的地址）
000000000000803c        movq    %rdx, %rcx 
000000000000803f        subq    $-0x80, %rcx # %rdx存储第三个参数 （&__NSConcreteStackBlock+0x80）
0000000000008043        leaq    0x4522a5(%rip), %rsi    ## literal pool for: "__NSStackBlock__"
# # %rsi 存储第二个参数 (字符串"__NSStackBlock__")
000000000000804a        movq    %rax, %rdi # # rdi 寄存器存储第一个参数(即上一个函数的返回值)
```

```c
Class __NSStackBlock = _objc_lookUpClass("__NSStackBlock");
objc_initializeClassPair_internal(__NSStackBlock, "__NSStackBlock__", &__NSConcreteStackBlock, &__NSConcreteStackBlock+0x80);

// objc_allocateClassPair(Class  _Nullable __unsafe_unretained superclass, const char * _Nonnull name, size_t extraBytes)
```

即在`__NSConcreteStackBlock`地址处，创建`__NSStackBlock__`类，继承自 `__NSStackBlock`，即`__NSConcreteStackBlock`是`__NSStackBlock__`类对象的首地址。

#### Block的flags 和invoke

```c
// 32位的flags存储了引用计数[1-16]位、block内存布局，block状态 具体可参考上面的枚举
volatile int32_t flags; // contains ref count
```

> The invoke function pointer is set to a function that takes the Block structure as its first argument and the rest of the arguments (if any) to the Block and executes the Block compound statement.
>
> [摘自Clang文档](https://clang.llvm.org/docs/Block-ABI-Apple.html)

block 的 flags 有两个位 BLOCK_HAS_COPY_DISPOSE(1 << 25) / BLOCK_HAS_SIGNATURE(1 << 30) 分别表示这个 block 的 descriptor 有没有 Block_descriptor_2 和 Block_descriptor_3 这两个结构体

```c
// 第一个参数是block本身
void (*invoke)(void *block, ...); 
```

#### Runtime Helper Functions

```c
// 代码摘自libclosure-67/runtime.c

/*******************************************************

Entry points used by the compiler - the real API!

A Block can reference four different kinds of things that require help when the Block is copied to the heap.// Block可以引用四中类型的对象，当拷贝到堆上时，需要copy/dispose辅助函数的支持
1) C++ stack based objects // C++ 栈上对象
2) References to Objective-C objects // OC对象
3) Other Blocks // 其他block
4) __block variables // __block修饰的变量

In these cases helper functions are synthesized by the compiler for use in Block_copy and Block_release, called the copy and dispose helpers.  The copy helper emits a call to the C++ const copy constructor for C++ stack based objects and for the rest calls into the runtime support function _Block_object_assign.  The dispose helper has a call to the C++ destructor for case 1 and a call into _Block_object_dispose for the rest.
// 辅助函数由编译器生成,会在Block_copy和Block_release中调用，copy辅助方法想c++栈上对象发起了copy构造方法的调用和一些其他的调用，dispose辅助方法调用C++析构方法并调用_Block_object_dispose

The flags parameter of _Block_object_assign and _Block_object_dispose is set to
    * BLOCK_FIELD_IS_OBJECT (3), for the case of an Objective-C Object,
    * BLOCK_FIELD_IS_BLOCK (7), for the case of another Block, and
    * BLOCK_FIELD_IS_BYREF (8), for the case of a __block variable.
If the __block variable is marked weak the compiler also or's in BLOCK_FIELD_IS_WEAK (16)

So the Block copy/dispose helpers should only ever generate the four flag values of 3, 7, 8, and 24.

When  a __block variable is either a C++ object, an Objective-C object, or another Block then the compiler also generates copy/dispose helper functions.  Similarly to the Block copy helper, the "__block" copy helper (formerly and still a.k.a. "byref" copy helper) will do a C++ copy constructor (not a const one though!) and the dispose helper will do the destructor.  And similarly the helpers will call into the same two support functions with the same values for objects and Blocks with the additional BLOCK_BYREF_CALLER (128) bit of information supplied.

So the __block copy/dispose helpers will generate flag values of 3 or 7 for objects and Blocks respectively, with BLOCK_FIELD_IS_WEAK (16) or'ed as appropriate and always 128 or'd in, for the following set of possibilities:
    __block id                   128+3       (0x83)
    __block (^Block)             128+7       (0x87)
    __weak __block id            128+3+16    (0x93)
    __weak __block (^Block)      128+7+16    (0x97)
        
_Block_object_assign(或者_Block_object_dispose)会根据flags的值来决定调用相应类型的copy helper(或者dispose helper)
********************************************************/
```

```c
void _Block_object_assign(void *destAddr, const void *object, const int flags);
void _Block_object_dispose(const void *object, const int flags);
```

#### Block的signature

signature 就是表示这个block 返回类型/参数类型的数据，类似这样的：”i8@?@8″。

```
'v'  :   void类型
'@'  :   一个id类型的对象
':'  :   对应SEL 

```



#### 实践

```c
// main.m
typedef id(^TestBlock)(int);

int main(int argc, const char * argv[]) {
    int captureValue = 3;
    TestBlock block = ^id(int param){
        captureValue = 4;
        return nil;
    };
    block(3);
    return 0;
}
```

```shell
clang -rewrite-objc main.m
```

```c
// main.cpp
int main(int argc, const char * argv[]) {
    int captureValue = 3;
     // 调用block构造方法，创建block对象
    TestBlock block = ((id (*)(int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, captureValue));
   // 简化一下即 TestBlock block =  &__main_block_impl_0(__main_block_func_0,&__main_block_desc_0_DATA,captureValue)
    // 调用block对象FuncPtr函数指针指向的函数
    ((id (*)(__block_impl *, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 3);
    // 即 block->FuncPtr(block,3);
    return 0;
}

// 在来看block的定义
struct __main_block_impl_0 {
  // struct __block_impl impl;
  struct __block_impl {
      void *isa;
      int Flags;
      int Reserved;
      void *FuncPtr;
  }
  // struct __main_block_desc_0* Desc;
  static struct __main_block_desc_0 {
    size_t reserved; // 0
    size_t Block_size; // sizeof(struct __main_block_impl_0)
 }
  int captureValue; // 捕获的外部局部变量 captureValue
  
  // 结构体构造方法
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _captureValue, int flags=0) : captureValue(_captureValue) {
    impl.isa = &_NSConcreteStackBlock; // 栈block
    impl.Flags = flags; 
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// 再看构造法的第一个参数 函数指针
static id __main_block_func_0(struct __main_block_impl_0 *__cself, int param) {
        int captureValue = __cself->captureValue; // bound by copy
        NSLog(@"%d",captureValue); // 已简化
        return __null;
  }

// 再看构造法的第二个参数 
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)}

// 最后一个参数即捕获的变量 captureValue

```

#### 引用

[深入研究Block捕获外部变量和__block实现原理](https://halfrost.github.io/2016/08/%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6Block%E6%8D%95%E8%8E%B7%E5%A4%96%E9%83%A8%E5%8F%98%E9%87%8F%E5%92%8C__block%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86/)

[如何动态调用 C 函数](http://blog.cnbang.net/tech/3219/)

[The relationship with _NSConcreteMallocBlock and NSMallocBlock?](https://stackoverflow.com/questions/38722359/the-relationship-with-nsconcretemallocblock-and-nsmallocblock)

[Objective-C runtime 拾遗 （三）——Block冷知识](https://segmentfault.com/a/1190000006823535)