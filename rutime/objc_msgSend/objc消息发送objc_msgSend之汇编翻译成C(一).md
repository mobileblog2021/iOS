

# 详细分析objc_msgSend

> 文章中的代码都是在 `x86_64` 架构下运行的，arm架构的代码大同小异。
>
> 这里分析的版本是objc4-706

## 写在前面

这篇文章是对objc_msgSend分析的第一篇文章，会把objc_msgSend方法中的汇编部分翻译成c代码。详细代码参见于[coding](https://git.coding.net/terrylxt/SuningEBuy.git)



## 准备工作

```objective-c

// LTObject.h
@interface LTObject : NSObject

- (void)hello;

@property (strong,nonatomic) NSString *world;

@end

// LTObject.m
@implementation LTObject

- (void)hello {
    NSLog(@"Hello");
}

@end

// main.m
int main(int argc, const char * argv[]) {
    
    LTObject *object = [[LTObject alloc] init];
    // [object hello]; 会被翻译成如下代码
    ((void (*)(id, SEL))(void *) objc_msgSend)((id)object, @selector(hello));
    
    return NSApplicationMain(argc, argv);
}
```

这里创建了一个自定义类，方便我以后扩展更多的测试；并在入口方法中添加测试代码。

## 进入正题

因为各种架构下的汇编代码是不一样的，因为寄存器的名称，大小和汇编指令是不相同的，所以objc_msgSend中汇编代码对应的多个汇编文件中，

```c
// .s 是汇编代码文件的后缀
objc-msg-x86_64.s  
objc-msg-arm.s     
objc-msg-simulator-i386.s
```

我们先来看一看原始的objc_msgSend源码,来自[objc-msg-x86_64.s](https://opensource.apple.com/source/objc4/objc4-706/runtime/Messengers.subproj/objc-msg-x86_64.s.auto.html),并简要的分析一下，后面会详细分析

```assembly
 
    ENTRY _objc_msgSend ; 定义方法 一个方法的入口，对应最底部的END_ENTRY
	UNWIND _objc_msgSend, NoFrame
	MESSENGER_START ; 前面以上都是汇编的伪指令，最终不会翻译成执行代码

	NilTest	NORMAL ; 检测receiver是否为nil ，如果为nil，立即返回

	GetIsaFast NORMAL		; r10 = self->isa 获得isa
	CacheLookup NORMAL, CALL ; calls IMP on success 在缓存中查找方法，如果找到则调用

	NilTestReturnZero NORMAL ; 这里对应一个标签，如果上面检测receiver是nil，就会跳转到这个标签，立即返回

	GetIsaSupport NORMAL ; 获得isa的一些辅助代码，会被上面的GetIsaFast调用

  ; cache miss: go search the method lists 如果在class的缓存没有找到，则会查找类的方法列表
LCacheMiss:
	; isa still in r10
	MESSENGER_END_SLOW
	jmp	__objc_msgSend_uncached ;跳转到__objc_msgSend_uncached中查找方法并调用

	END_ENTRY _objc_msgSend
```

因为要调试汇编代码，源码中为了汇编代码的复用使用了大量的汇编伪指令宏，类似C语言中的宏，同样的我们不能使用lldb调试宏对应的代码。

```assembly
;语法
.macro 宏名称
   ... ;宏定义语句
.endmacro 

; 举个例子 调用示例：GetIsaFast STRET
.macro GetIsaFast  
.if $0 != STRET ; $0,$1 依次代表第一个参数，第二个参数
	testb	$$1, %a1b ; 在A&AT格式下立即数之前要加$ ,所以在.marco中加两个$表示立即数
	...
.else
	testb	$$1, %a2b
     ...
.endif	
.endmacro
```

为了能使用lldb调试汇编代码，只能一个个把这些宏指令替换成对应的汇编代码，以方便获得更多的信息

替换后，代码变得极长，这里分在几个代码快上，但都是连续的

#### 前面的伪指令

```assembly
   
   ; ENTRY _objc_msgSend 
	.text  ; 表示这些代码存放在代码段
	.globl	_objc_msgSend ; 全局标识符
	.align	6, 0x90  ; 对齐方式
    _objc_msgSend:  ; 记录一个方法的入口

    ;UNWIND _objc_msgSend, NoFrame 
    .section __LD,__compact_unwind,regular,debug
    .quad _objc_msgSend
    .set  LUnwind_objc_msgSend, LExit_objc_msgSend - _objc_msgSend
    .long LUnwind_objc_msgSend
    .long NoFrame
    .quad 0	  ; no personality
    .quad 0   ; no LSDA
    .text

     ;MESSENGER_START
    4:
    .section __DATA,__objc_msg_break
    .quad 4b
    .quad ENTER
    .text
    
```

#### NilTest

```assembly
  ;判断第一个参数是否为nil 如果检测方法的接受者是nil，那么系统会自动clean并且return,这一点就是为何在OC中给  nil发送消息不会崩溃的原因。
 ;NilTest	NORMAL 
 
    testq	%rdi, %rdi 
    PN
    jz	LNilTestSlow_f  ; #define LNilTestSlow_f 	7f  即 jz 7f 
```

rdi寄存器存放方法的第一个参数,即receiver，rsi寄存器存放方法的第一个参数,即selector，更多寄存器的用法参见[TODO]；

testq指令：将两个参数进行与运算，结果不会保存到目标寄存器（这里是第二个rdi）中，只会修改部分标志寄存器，例如ZF (零标志/Zero Flag)

这里用于判断receiver指针是否0（因为nil对应指针地址是0x0)，两个相等的数进行与运算，只有当这个数是0的时候，结果才能为0,才能把ZF置为1 （1即是0，0代表非0）

jz （jump is zero):如果ZF中的值是1，即上一次运算的结果为0，则跳转，否则继续向下执行

```c
/********************************************************************
 * Harmless branch prefix hint for instruction alignment 用于优化分支
 ********************************************************************/
	
#define PN .byte 0x2e
```

jz 7f 中f （forward）代表，向前找对应的地址标号，b （backward）向后找对应的地址标号

```assembly
    ;NilTestReturnZero NORMAL
    .align 3
    ;NilTestSlow: ;#define LNilTestSlow 	7
    ;ZeroReturn
    xorl	%eax, %eax  ;清空eax 用于返回值
    xorl	%edx, %edx  ;清空edx 第一个参数
    xorps	%xmm0, %xmm0 ;sse提供了xmm寄存器，处理128位打包数据,xmm一组8个128位的寄存器，分别名为xmm0-xmm7，sse构架提供对打包单精度浮点数的SIMD（Streaming SIMD Extension, SSE)支持
    xorps	%xmm1, %xmm1 ;http://fancymore.com/reading/assembler-sse-instruct.html
```

这里使用异或指令清零了四个相关的寄存器， 把eax（rax的低32位）用于存放返回值的寄存器清零，即return nil

综上，翻译为c代码

```objective-c
if(!self){
  return nil;
}
```

#### GetIsaFast 获得self的isa结构体，并放到r10寄存器中

```assembly
   ;GetIsaFast NORMAL		// r10 = self->isa  // 找到对象的isa 存放到r10寄存器中

    testb	$1, %dil  
    PN
    jnz	LGetIsaSlow_f  // #define LGetIsaSlow_f 	9f 
    movq	$0x00007ffffffffff8, %r10 
    andq	(%rdi), %r10
    LGetIsaDone:  ; #define LGetIsaDone 	8
```

`testb	$1, %dil `   细心的同学，可能主要到上面用的是tesq，这里用的是testb，主要区别是操作的位数不同，这里testb代表1byte=8bit，dil是rdi寄存器的低8位；想要了解更多,请参考[TODO]，

这句命令的意思是判断第一个参数self指针的最低位是否为1；’

`jnz	LGetIsaSlow_f`  jnz (jump if not zero)：即如果self指针的第一位不为1则跳转到前面的地址标号9处，否则继续执行；

` movq	$0x00007ffffffffff8, %r10 `  把立即数0x00007ffffffffff8存入r10寄存器；

`andq	(%rdi), %r10`   （%rdi）代表rdi寄存中的地址指向的值，即指针指向的对象 ，

即 self.isa.bits & 0x00007ffffffffff8；



**GetIsaSlow** 用于从Tagged Pointer 中获得class 地址（包括Extended Tagged Pointer），这里稍微提一下，Tagged Pointer 指针并不指向一个对象，而是把数据和数据对应的class（指针的前4位存放在class数组中的索引） 放在指针中，从而在定义一些较小的数据时，可以避免在堆中分配内存，从而达到节省内存和提高执行效率的目的，如果想了解其更多细节，请[Tagged Pointer Strings](https://www.mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html)，及该博客中相关的其他文章。

**Extended Tagged Pointer** 是使用更多的[4-11]位，来存放更多的Tagged Pointer Class ，但对应的数据长度会缩水。

```assembly
   ; GetIsaSupport NORMAL

    LGetIsaSlow:
    movl	%edi, %r11d   ; r11=ptr
    andl	$0xF, %r11d   ;  ptr && &0xF  取self的前四位
    cmp	$0xF, %r11d ; 如果前四位都是1, 则是 extended tagged pointer
    je	1f   ; jump if equal
    ; basic tagged 普通的Tagged Pointer,前四位存放class对应的索引，其余的存放数据
    ; RIP相对寻址 r10=_objc_debug_taggedpointer_classes ，即对应官方注释中的read isa from table
    ; 这个是存放tagged Pointer class 的数组的起始地址 r11 存放的是指针的[0-3]位
    ; r11 对应着tagged Pointer class的偏移
    ; 每个指针占用8个字节 所以对应的tagged Pointer class地址是=起始地址+偏移*数组成员大小
    leaq _objc_debug_taggedpointer_classes(%rip), %r10   ;rip寄存器存放着下一条执行指令的地址
    movq (%r10, %r11, 8), %r10 ;  cls=r10=_objc_debug_taggedpointer_classes[r11]
    jmp	LGetIsaDone_b  ; #define LGetIsaDone_b 8b
    1:
    ; extended tagged 同理 第[4-11]位 存放class对应的索引，其余的存放数据
    movl	%edi, %r11d ; r11d = ptr
    shrl	$4, %r11d  ; (ptr >> 4) & 0xff
    andl	$0xFF, %r11d ; r11d=ptr的[4-11]位
    leaq	_objc_debug_taggedpointer_ext_classes(%rip), %r10    %r10=&_objc_debug_taggedpointer_ext_classes
    movq	(%r10, %r11, 8), %r10 ; read isa from table
    jmp	LGetIsaDone_b
```

` leaq	_objc_debug_taggedpointer_classes(%rip), %r10 `这里涉及到rip相对寻址，即数据的相对寻址相对于当前指令地址（PC）的一种寻址方式，可参考[x64下PIC的新寻址方式：RIP相对寻址](https://www.polarxiong.com/archives/x64%E4%B8%8BPIC%E7%9A%84%E6%96%B0%E5%AF%BB%E5%9D%80%E6%96%B9%E5%BC%8F-RIP%E7%9B%B8%E5%AF%B9%E5%AF%BB%E5%9D%80.html)和[如何理解DLL不是地址无关的？DLL与ELF的对比分析](https://www.polarxiong.com/archives/%E5%A6%82%E4%BD%95%E7%90%86%E8%A7%A3DLL%E4%B8%8D%E6%98%AF%E5%9C%B0%E5%9D%80%E6%97%A0%E5%85%B3%E7%9A%84-DLL%E4%B8%8EELF%E7%9A%84%E5%AF%B9%E6%AF%94%E5%88%86%E6%9E%90.html)

在这里这句指令的作用是把获得在数据段定义的_objc_debug_taggedpointer_classes首地址。

下面是_objc_debug_taggedpointer_classes和_objc_debug_taggedpointer_ext_classes在[objc-msg-x86_64.s](https://opensource.apple.com/source/objc4/objc4-706/runtime/Messengers.subproj/objc-msg-x86_64.s.auto.html)中的定义。

```assembly
	.data  ; 告诉编译器，在数据段
	.align 3
	.globl _objc_debug_taggedpointer_classes ;告诉编译器 _objc_debug_taggedpointer_classes可在外部文件中访问
_objc_debug_taggedpointer_classes:  ;示例：objc_debug_taggedpointer_classes[_OBJC_TAG_SLOT_COUNT*2]
	.fill 16, 8, 0   ; Tagged pointer 的[0-3]存放class类型 类型 2^4=16 每个class指针大小是8个字节
	.globl _objc_debug_taggedpointer_ext_classes // objc_debug_taggedpointer_ext_classes[_OBJC_TAG_EXT_SLOT_COUNT]; #define _OBJC_TAG_EXT_SLOT_COUNT 256
_objc_debug_taggedpointer_ext_classes:
	.fill 256, 8, 0 ; Texended pointer 的[4-11]存放class类型 类型 2^8=256 每个class指针大小是8个字节
```

TODO:这里遗留一个问题，这些数据在什么时候填充的，问题会在以后的源码阅读中，慢慢揭开。

翻译成c代码（这里部分参考官方源码）

```objective-c
#   define _OBJC_TAG_MASK 1
#   define ISA_MASK        0x00007ffffffffff8ULL

static inline bool
isTaggedPointer(const void *ptr)
{
    // 如果指针的第一位是1，那么则是TaggedPointer ，对象的指针都是64位对齐，第一位都是0
    return ((intptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}

static inline bool
isExtTaggedPointer(const void *ptr)
{
    //  define _OBJC_TAG_EXT_MASK 0xfULL
    return ((uintptr_t)ptr & _OBJC_TAG_EXT_MASK) == _OBJC_TAG_EXT_MASK;
}

static inline Class 
getIsaSlow(id receiver){
    uintptr_t ptr = (uintptr_t)receiver;
    if (isExtTaggedPointer((__bridge const void *)(receiver))) {
        // #define _OBJC_TAG_EXT_SLOT_SHIFT 4
        // #define _OBJC_TAG_EXT_SLOT_MASK 0xff
        // 取 4-11位
        uintptr_t slot = (ptr >> 4) & 0xff;
        return objc_debug_taggedpointer_ext_classes[slot];
    } else {
        // 取地址二进制的前四位（0-3位）  #define _OBJC_TAG_SLOT_MASK 0xf 第一位用来确定是否是TaggedPointer，后三位用来判断是TaggedPointer的数据类型
        uintptr_t slot =(ptr >> 0) & 0xf;
        return objc_debug_taggedpointer_classes[slot];
    }
}

static inline Class 
GetIsaFast(self){
   Class cls=nil;
   if ( isTaggedPointer((__bridge const void *)(self)) ) {
        // 获得Tagged Pointer 指针中的对应的class
        cls=getIsaSlow(self);
   }else{
       // 获得普通类对象的class 存放于 union isa中bits的[4-47]位，及对应isa中的shiftcls
       // 这是64位的情况，32位则class直接放在union isa 的cls中
       objc_object * object=(__bridge objc_object *)self;
       cls=(__bridge Class)((void *)(object->isa.bits & ISA_MASK));
  }
  return cls;
}
```
#### CacheLookup NORMAL, CALL  在缓存中查找IMP，如果找到则调用

 涉及到的结构体

```c++
struct objc_class : objc_object {
    // 指向元类，来保证无论是类还是对象都能通过相同的机制查找方法的实现 
    // 这里是类方法调用，实例方法是通过对象的isa在类中获取方法的实现
    // Class ISA;              
    Class superclass;          // 指向父类，用来查找继承的方法
    cache_t cache;             // formerly cache pointer and vtable 缓存
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags  在其中查找对应方法的实现
};

typedef uint32_t mask_t; 

struct bucket_t {
    cache_key_t _key; // key=(uintptr_t)sel;
    IMP _imp; 
};

struct cache_t {
    // hash表,因为散列表检索起来更快, 每一个bucket_t存储一个方法的缓存，有空的
    struct bucket_t *_buckets; 
    // 是当前能达到的最大index（从0开始的），所以缓存(hash表)的size（total）是mask+1
    mask_t _mask; 
    //已占用的大小 ,被占用的槽位，因为缓存是以散列表的形式存在的，
    // 所以会有空槽，而occupied表示当前被占用的数目
    mask_t _occupied; 
    // capacity=_mask ? _mask+1 : 0; //分配的大小，即当前的容量
}
```

CacheLookup 汇编源码

```assembly
    ; CacheLookup NORMAL, CALL	// calls IMP on success  查找成功则调用
    ; rsi寄存器存放函数的第二个参数这里是_cmd
    movq	%rsi, %r11		 ; r11 = rsi (即_cmd)  rsi = 0x00000001005faa10  "hello"
    ; r10存放的是class（参见上一步GetIsaSlow）, 24(%r10)表示class指向的对象地址向下偏移24字节,得到class->cache.mask
    ; 为什么是24呢，看上面的结构体定义,从class对象首地址开始+ISA+superclass+cache_t._buckets 偏移
    ; 即   class->cache.mask=class+ISA(8byte)+superclass(8byte)+_buckets(8byte)
    andl	24(%r10), %r11d	 ; r11d=_cmd & class->cache.mask 
    ; ; r11 = offset = (_cmd & class->cache.mask) << 4 详细分析见下面问题1
    shlq	$4, %r11 ; offset* 2^4  因为 16=sizeof(bucket_t)
    ; 同上 class->cache._buckets=16(%r10)
    ; r11 = class->cache.buckets + offset
    ; r11 存放的的找到的bucket的首地址，也是bucket->sel的首地址
    addq	16(%r10), %r11	

    cmpq	(%r11), %rsi  ; if (bucket->sel != _cmd)
    jne 	1f	;  不是要找的imp,进行循环 scan more
    
    ;  CacheHit NORMAL, CALL  // call or return imp
    ; CacheHit must always be preceded by a not-taken `jne` instruction
    ; in order to set the correct flags for _objc_msgForward_impcache.

    ; r11 = found bucket
    MESSENGER_END_FAST ;  记录方法结束时的状态 ，方便调试 对应的有下面的MESSENGER_END_SLOW

    jmp	*8(%r11) ; call imp  bucket->imp()

    1:
    ; loop
    cmpq	$1, (%r11) 
    ; if (bucket->sel <= 1) wrap or miss
    ; 如果bucket->sel=0 则是空的bucket；如果bucket->sel=1 4 则是标记结束的bucket
    ; 这会在后面的方法缓存策略中再次谈到，为什么有这样的对应关系
    ; 即 bucket->sel <= 1代表无效的缓存
    jbe	3f	; jump if blow or equal		

    addq	$16, %r11		; bucket++ 有效的缓存 但不是要找，则继续向后遍历
    2:
    cmpq	(%r11), %a2		; if (bucket->sel != _cmd)
    jne 	1b			;     scan more
    CacheHit NORMAL, CALL	; call or return imp 上面已经有过解释，这里不再赘述

    3:
    ; wrap or miss 在后面或者没有
    jb	LCacheMiss_f		;if (bucket->sel < 1) cache miss
    ;  wrap 在后面
    ; bucket->imp 里存放着第一个bucket的首地址
    movq	8(%r11), %r11	; bucket->imp is really first bucket 
    jmp 	2f

    ;Clone scanning loop to miss instead of hang when cache is corrupt.
    ;The slow path may detect any corruption and halt later.
    ; 为什么会有两份近似的循环代码和 bucket->imp 里存放着第一个bucket的首地址
    ; 详细分析见下面问题2

    1:
    ; loop
    cmpq	$1, (%r11)
    jbe	3f			; if (bucket->sel <= 1) wrap or miss

    addq	$16, %r11		; bucket++
    2:
    cmpq	(%r11), %a2		; if (bucket->sel != _cmd)
    jne 	1b			; scan more
    CacheHit NORMAL, CALL	; call or return imp

    3:
    ; double wrap or miss
    jmp	LCacheMiss_f
    
; cache miss: go search the method lists
LCacheMiss:
	; isa still in r10
	MESSENGER_END_SLOW 

    ; __objc_msgSend_uncached 会保存寄存器状态 并调用_class_lookupMethodAndLoadCache3
    ; 为什么要保存寄存器状态，详细分析见问题3
    ; IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
	jmp __objc_msgSend_uncached

	END_ENTRY _objc_msgSend
```

先看一下，缓存是怎么填充的，这方便我们理解为什么这样在缓存中查找对应的方法。

```c++
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{
    // ... 省略与这里无关的代码，以后会涉及   在空bucket不足时，扩增
    cache_key_t key = (cache_key_t)sel;
    // 根据key 找到空的bucket
    bucket_t *bucket = cache->find(key, receiver);
    if (bucket->key() == 0) cache->incrementOccupied();
    // 填充
    bucket->set(key, imp);
}

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

对应的C代码

```c++

static inline void
CacheLookup(id self,SEL _cmd,objc_class * cls){
    uintptr_t cmdptr=(uintptr_t)((const void*)(_cmd));
    // capacity = mask +1;
    // 把 _csm 指针地址和缓存的mask进行与运算 则结果 0<index<=mask，这会在问题1中详细分析
    uint32_t index = cmdptr & cls->cache._mask; 
    struct bucket_t *bucket=cls->cache._buckets[index]; 
    
    // 最优情况下，直接命中，执行imp，无需遍历 
    if (bucket->_key==cmdptr) {
        IMP imp=bucket->_imp;
        imp();
        return;
    }
   
    bool isEndBucketSearched=false; // 是否已经便利过结束标志bucket,在遍历一遍后，避免死循环
    // 在index 之后开始遍历缓存，最差的情况时，把所有的bucket遍历一遍
    // 多数情况下，遍历少量的部分bucket
    do{
        if(bucket->_key <=1){
            if (bucket->_key <1 || isEndBucketSearched) {
                // cache miss 没有在缓存中，找到对应的方法，去class的方法列表中中查找
                IMP imp=_class_lookupMethodAndLoadCache3(self,_cmd,(__bridge Class)cls);
                imp();
                return;
            }else{
                // bucket->_key =1，bucket imp 里存放是hash表的首地址 遍历到了
                bucket=(struct bucket_t *)bucket->_imp;
                isEndBucketSearched=true;
            }
        }else{
            // 向后查找 
            bucket++;
        }
    }while (bucket->_key!=cmdptr);
    
    
    IMP imp=bucket->_imp;
    imp();
}
```

 问题1：为什么根据SEL 在哈希表中查找对应的IMP时，要进行与操作 （_cmd & class->cache.mask）,并把与操作的结果作为index开始向后遍历？

这是为了降低查找缓存的复杂度；

```c++
// 分配新容量的bucket_t，并设置好最后的标志结束buckets 对应x86_64架构，简化后的代码
bucket_t *allocateBuckets(mask_t newCapacity)
{
    // Allocate one extra bucket to mark the end of the list.
    // 多分配一个bucket，用于标记列表的结束
    bucket_t *newBuckets = (bucket_t *)calloc(sizeof(bucket_t) * (cap + 1), 1);
    // 最后一个buckets
    bucket_t *end = cache_t::endMarker(newBuckets, newCapacity);
    end->setKey((cache_key_t)(uintptr_t)1); // 存放的不是SEL，而是1
    end->setImp((IMP)newBuckets);//存放的不是IMP，而是第一次或者扩容后的 class->cache._buckets的首地址
    return newBuckets;
}
```

在把SEL，IMP存入bucket时，通过 index=_cmd & mask,

- 保证了index∈=[0,mask],不会数组越界

- 有让三者产生了关系，在查找时通过 _cmd & mask 便可得到对应的index，最优情况下，遍无需遍历；

  如果在缓存填充时，index对应的bucket已经被其他方法填充了，则向后查找最近的空bucket；

  如果遍历到了最后的结束标志bucket，因为在buckets hash表的最后一个标志bucket，便可从中得到class->cache._buckets的首地址，从头开始遍历。

  因为在缓存填充时，都会检测被占用的bucket数量，如果 occupied > capacity /4*3,容量则会翻倍，所以永远不会有缓存被充满的情况发生。

  最差的极端情况下，会遍历已有的所有已填充的bucket；

  多数情况下，只会遍历少量的bucket；

问题2：为什么汇编代码中有两份近似的循环代码？

- 第一个循环代码中，对结束标志bucket进行了处理，第二个则无需处理
- Clone scanning loop to miss instead of hang when cache is corrupt.The slow path may detect any corruption and halt later. 即当cache是不合法的，只会出现缓存找不到的情况，不会使程序挂起。

问题3：在__objc_msgSend_uncached 方法中为什么要保存那么多的寄存器状态？

```assembly
     STATIC_ENTRY __objc_msgSend_uncached ; 定义 __objc_msgSend_uncached 开始 {
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves 

	; THIS IS NOT A CALLABLE C FUNCTION // 这不是一个可用调用的方法，因为依赖着外部数据
	; Out-of-band r10 is the searched class // r10中存放着在哪个类中搜索方法
	; MethodTableLookup NORMAL	// r11 = IMP
    push	%rbp   ; 把上一个栈帧的栈底指针入栈
    mov	%rsp, %rbp ; 设置当前栈帧的栈底指针

    sub	$0x80+8, %rsp  ; +8 for alignment 开辟 0x80+8大小的占空间
    
    ; sse提供了xmm寄存器，处理128位打包数据,xmm一组8个128位的寄存器，分别名为xmm0-xmm7，
    ; sse构架提供对打包单精度浮点数的SIMD（Streaming SIMD Extension, SSE)支持
    ; 把 xmm0-xmm7,rax，6个参数寄存器 入栈，入栈后的效果见下方
    ; 寄存器 %rax 扮演了一个隐藏的参数。它用于变参的调用，并保存传入的向量寄存器（vector registers）的数量，用于被调用的函数可以正确的准备变参列表。以防目标函数是个变参的方法
    ; 用于传入浮点类型参数的寄存器 %xmm 也应该被保存
    ; mov 和 push 指令的差别  push %rax 等价于 sub $8,%rsp; movq %rax,(%rsp
    movdqa	%xmm0, -0x80(%rbp)
    push	%rax			// might be xmm parameter count
    movdqa	%xmm1, -0x70(%rbp)
    push	%rdi
    movdqa	%xmm2, -0x60(%rbp)
    push	%rsi
    movdqa	%xmm3, -0x50(%rbp)
    push	%rdx
    movdqa	%xmm4, -0x40(%rbp)
    push	%rcx
    movdqa	%xmm5, -0x30(%rbp)
    push	%r8
    movdqa	%xmm6, -0x20(%rbp)
    push	%r9
    movdqa	%xmm7, -0x10(%rbp)

    ; _class_lookupMethodAndLoadCache3(receiver, selector, class)

    ; receiver already in rdi
    ; selector already in rsi

    movq	%r10, %rdx
    call	__class_lookupMethodAndLoadCache3

    ; 方法的返回结果(IMP),按规定放在RAX寄存中  IMP is now in %rax
    movq	%rax, %r11
   
    ; 出栈 
    movdqa	-0x80(%rbp), %xmm0
    pop	%a6
    movdqa	-0x70(%rbp), %xmm1
    pop	%a5
    movdqa	-0x60(%rbp), %xmm2
    pop	%a4
    movdqa	-0x50(%rbp), %xmm3
    pop	%a3
    movdqa	-0x40(%rbp), %xmm4
    pop	%a2
    movdqa	-0x30(%rbp), %xmm5
    pop	%a1
    movdqa	-0x20(%rbp), %xmm6
    pop	%rax
    movdqa	-0x10(%rbp), %xmm7

    cmp	%r11, %r11		; set eq for nonstret forwarding

    ; LEAVE is the counterpart to ENTER. The ENTER instruction sets up a stack frame by first       pushing RBP onto the stack and then copies RSP into RBP, 
    ; so LEAVE has to do the opposite, i.e. copy RBP to RSP and then restore the old RBP from the stack.
    ; leave 和enter 成对使用，enter指令设置栈帧，push %rbp     movq %rsp %rbp
    ; leave 相反， movq %rbp %rsp     pop %rbp
    leave

	jmp	*%r11			; goto *imp 调用IMP

	END_ENTRY __objc_msgSend_uncached ; 定义 __objc_msgSend_uncached 开始 
```

入栈后的效果

```c
/*                              | rpb |                    高地址
 *      ---- rbp 0xf0+8
 *      ---(-0x10(%rbp))        | xmm7 |
 *      ---(-0x20(%rbp))        | xmm6 |
 *      ---(-0x30(%rbp))        | xmm5 |
 *      ---(-0x40(%rbp))        | xmm4 |
 *      ---(-0x50(%rbp))        | xmm3 |
 *      ---(-0x60(%rbp))        | xmm2 |
 *      ---(-0x70(%rbp))        | xmm1 |
 *      ---0x70+8(即-0x80(%rbp))  | xmm0 |
 *      ---0x70                 | rax  |
 *      ---0x70-8               | rdi  |
 *      ---0x70-16              | rsi  |
 *      ---0x70-24              | rdx  |
 *      ---0x70-32              | rcx  |
 *      ---0x70-40              | r8   |
 *      ---(rsp)0x70-48         | r9   |                    低地址
 */
```

我们知道可以使用C语言的宏（**va_start**等,但比较慢），实现可变参数，那么objc_msgSend是怎么实现可变参数的呢？

即从objc_msgSend开始，到调用找到的IMP ，这段时间保持参数寄存器的状态，不要被覆盖，这样就等价于配置好参数寄存器后，直接调用IMP。

即任何传给 objc_msgSend 的参数，最终都会成为 IMP 的参数。 IMP 的返回值成为了最开始被调用的方法的返回值。

因为没有方法可以将传入 C 函数的泛参（generic parameters）传给另一个函数。 你可以使用变参，但是变参和普通参数的传递方法不同，而且慢，所以这不适合普通的 C 参数。

## 总结

- 如果在缓存中找到IMP 直接调用，流程走完
- 如果没有在缓存中找到IMP，则调用C方法并调用_class_lookupMethodAndLoadCache3从class 及集成链上查找，这是下一篇文章所要谈及的。
- 剩余的汇编部分，会在使用到地方，进行分析