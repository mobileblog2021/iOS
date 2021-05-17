系统提供的NSLog提供的功能太简单了，我们能不能扩展一下呢？我们除了关心输出结果以外还关心什么？当前NSLog所在的文件？当前的方法？当前的行号？输出结果的优先级？和其他Log的隔离？

现有的工程中已经有很多NSLog了，我们可以通过系统内置的宏（Builtin Maros）,对其无痛替换，只要把下面代码加载pch文件中就可以了。

```objective-c
#if !defined(NSLog) || !defined(NSLogD) || !defined(NSLogW)               // 避免重复定义
    #define NSLog(format, ...)  kLTPrivateLog(@"💙",format,##__VA_ARGS__) // 💙 debug
    #define NSLogE(format, ...) kLTPrivateLog(@"💜",format,##__VA_ARGS__) // 💜 error
    #define NSLogW(format, ...) kLTPrivateLog(@"💛",format,##__VA_ARGS__) // 💛 warning
    // 私有宏  __FILE__：类文件 __LINE__：所在行 __func__：方法
    #define kLTPrivateLog(priority,format, ...)  LTPrivateLog(priority,__FILE__,__func__,__LINE__,format,##__VA_ARGS__)

    // NS_FORMAT_FUNCTION(5,6) 使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配 NSLog(@"测试：%s",3); 这样则会给出警告
    static inline NS_FORMAT_FUNCTION(5,6) void LTPrivateLog(NSString *priority,const char *file,const char *function,int line,NSString *format, ...){
    #if !defined(__OPTIMIZE__) // release模式 优化选项开启时，关闭log输出
        do {
            va_list lt_args;
            va_start(lt_args, format);
            fprintf(stderr, "<%s : %d> %s\n",
                    [[[NSString stringWithUTF8String:file] lastPathComponent] UTF8String],
                    line, function); // 所在文件+行号+方法名
            (NSLogv)(([priority stringByAppendingString:format]),(lt_args)); // 结果
            fprintf(stderr, "--------------------------------------------------\n");// 分割
            va_end(lt_args);
        } while (0);
    #endif
    }

#endif
```

最后，输出结果是这样的

```
<main.m : 22> main
2016-12-16 10:58:22.631 MacroTest[5652:473246] 💙正常的调试
--------------------------------------------------
<main.m : 22> main
2016-12-16 10:58:22.631 MacroTest[5652:473246] 💛警告
--------------------------------------------------
<main.m : 22> main
2016-12-16 10:58:22.631 MacroTest[5652:473246] 💜错误
--------------------------------------------------
```

这里写成c方法，是为了方便调试和扩展，也可以写成宏

```objective-c

// 💙 debug
#define NSLog(format, ...)  LTPrivateLog(@"💙",format,##__VA_ARGS__)
// 💜 error
#define NSLogE(format, ...) LTPrivateLog(@"💜",format,##__VA_ARGS__)
// 💛 warning
#define NSLogW(format, ...) LTPrivateLog(@"💛",format,##__VA_ARGS__) 

#define LTPrivateLog(priority,format, ...)                                                     \
        do {                                                                                   \
            fprintf(stderr, "<%s : %d> %s\n",                                                  \
            [[[NSString stringWithUTF8String:__FILE__] lastPathComponent] UTF8String],         \
            __LINE__, __func__);                                                               \
            (NSLog)(([priority stringByAppendingString:format]), ##__VA_ARGS__,nil);           \
            fprintf(stderr, "--------------------------------------------------\n");           \
        } while (0)
```

其中NSLog最后一个参数nil是为了消除Format string is not a string literal 警告



好了，终于引出了我们今天的主题 Builtin Macros,标准的预定义宏都是双下划线开头



## Builtin Macros

```c
// 当前文件的绝对路径  /Users/terry/WorkSpace/ios/MacroTest/MacroTest/main.m
__FILE__ (char *)
  
// 在当前文件中的行数  
__LINE__ (int)
  
// 当前代码段所在的函数名 
__func__ & __FUNCTION__ (char * )

// 预处理器预处理操作的时间 char *
__DATE__ & __TIME__   (char *)
  
// 从0递增的有序整数
__COUNTER__ (int)
  
// 命令行的入口文件的绝对路径 （例如main）
__BASE_FILE__ (char *)
  
// 当前文件被引用的深度，main文件时该值为0 
__INCLUDE_LEVEL__ (int)
 
// iOS开发中 release模式通常会定义 __OPTIMIZE__，debug模式不会 (和Build Setting里的预定义宏DEBUG 相反)
#ifdef __OPTIMIZE__
 
示例
#ifdef __OPTIMIZE__    //(or #ifndef DEBUG)
     printf("release模式\n");
#else
     printf("debug模式\n");
#endif

// 源码文件最后修改时间
__TIMESTAMP__ (char *)

// 表示严格符合一些ISO C 和ISO C++ 版本的标准： 
#ifdef __STRICT_ANSI__
  
// 返回当前编译器是否支持clang GNU
#ifdef __clang__ & #ifdef __GNUC__
  
// 返回当前Clang/GNU(GCC)的主版本号
__clang_major__ & __GNUC__ (int)
  
// 返回当前Clang的次版本号
__clang_minor__ & __GNUC_MINOR__  (int)
  
// 返回当前Clang的补丁版本号
__clang_patchlevel__ &  __GNUC_PATCHLEVEL__ (int)
 
// 返回当前Clang的完整的版本号 8.0.0 (clang-800.0.42.1)  gcc-4.2.1
__clang_version__ (char *)

// 判断是否使用了Objective-C 编译器 用于测试一个.h文件是被C编译器 还是 Objective-C编译器编译
__OBJC__
 
// 当预处理汇编语言 该宏被定义为1
#ifdef __ASSEMBLER__
 
// 表示编译器符合ISO C标准
#ifdef __STDC__

// C标准版本   c99=gnu99=199901 c11=gnu11=201112
__STDC_VERSION__ (long)
  
// 表示宿主环境(hosted environment下任何的标准库可用, main函数返回一个int值,典型例子是除了 内核以外几乎任何的程式  
#ifdef __STDC_HOSTED__
  
```

## Builtin Functions

#### __builtin_assume( true expression )

切换成release模式，即开启优化选项

```c
int x=1;
if (x == 1) {
  printf("live code\n");
} else {
  printf("dead code\n");
}
// 优化编译器会移除else 变成如下代码
printf("live code\n");

// 添加  __builtin_assume
int x = 1;
__builtin_assume(x != 1);
if (x == 1) {
  printf("live code\n");
} else {
  printf("dead code\n");
}
// 优化编译器会移除if 变成如下代码 (经管这样是不正确的)
printf("dead code\n");
```

那这在什么样的场景下会使用呢？

考虑上面的`if/else`被置于一个内联方法中，再具体的使用场景中，程序员可能知道x是一个具体的值，但是编译器不知道，所以就有了多余的if判断

```objective-c

static inline void test(int a){
    if (a>0) {
       printf("a>0");
    }else {
       printf("a<=0");
    } 
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 假如在当前场景下，a>0 恒成立,这样便可优化掉内敛方法中if判断
        int a=1;
        __builtin_assume(a>0);
        test(a);
    }
    return 0;
}
```



### `__builtin_bitreverse8` 

逆转二进制值整数顺序 例如`0b10110110` 变成 `0b01101101`

```c
uint8_t rev_x = __builtin_bitreverse8(x);
uint16_t rev_x = __builtin_bitreverse16(x);
uint32_t rev_y = __builtin_bitreverse32(y);
uint64_t rev_z = __builtin_bitreverse64(z);
```



### `__builtin_unreachable`

__builtin_unreachable 表示当前代码不可到达，尽管编译器认为在某些情况下可以到达。这可用于编译器优化代码和消除警告，

```c
// __attribute__((noreturn)) 用来告诉编译器，此方法没有返回值
void myabort(void) __attribute__((noreturn)); 
void myabort(void) {
  asm("int3");
  __builtin_unreachable();
}
```

例如以上代码， 编译器认为内联方法asm可能失败，如果移除__builtin_unreachable(); 编译器会给出警告 `Function declared 'noreturn' should not return`

### __sync_swap

交换内存中的整数或指针

```
int old_value = __sync_swap(&value, new_value);
```



###  __builtin_return_address(LEVEL)

1、gcc默认不支持__builtin_return_address(LEVEL)的参数为非0。好像只支持参数为0。
2、__builtin_return_address(0)的含义是，得到当前函数返回地址，即此函数被别的函数调用，然后此函数执行完毕后，返回，所谓返回地址就是那时候的地址。
3、__builtin_return_address(1)的含义是，得到当前函数的调用者的返回地址。注意是调用者的返回地址，而不是函数起始地址。



### 其他

```
// 返回右起第一个‘1’的位置
int __builtin_ffs (unsigned int x)
  
// 返回左起第一个‘1’之前0的个数
int __builtin_clz (unsigned int x)

// 返回右起第一个‘1’之后的0的个数
int __builtin_ctz (unsigned int x)
  
// 返回‘1’的个数
int __builtin_popcount (unsigned int x)
  
// 返回‘1’的个数的奇偶性 
int __builtin_parity (unsigned int x)
```

#### 

###小技巧 

列出Clang预定义宏（64位）

```shell
echo | clang -dM -E -
# or
clang -E -dM -x objective-c /dev/null

clang -E -dM -x objective-c -fobjc-arc /dev/null
```

```
#define OBJC_NEW_PROPERTIES 1
#define _LP64 1  // __LP64__ 64位处理器
/*
 This macro is set to an integer that represents the version number of
 the compiler. This lets you distinguish, for example, between compilers
 based on the same version of GCC, but with different bug fixes or features.
 Larger values denote later compilers.
 */
#define __APPLE_CC__ 6000
#define __APPLE__ 1  // This macro is defined in any Apple computer
#define __ATOMIC_ACQUIRE 2
#define __ATOMIC_ACQ_REL 4
#define __ATOMIC_CONSUME 1
#define __ATOMIC_RELAXED 0
#define __ATOMIC_RELEASE 3
#define __ATOMIC_SEQ_CST 5
#define __BIGGEST_ALIGNMENT__ 16
#define __BLOCKS__ 1 // 是否支持block ,否则就使用函数指针
#define __BYTE_ORDER__ __ORDER_LITTLE_ENDIAN__
#define __CHAR16_TYPE__ unsigned short
#define __CHAR32_TYPE__ unsigned int
#define __CHAR_BIT__ 8
#define __CONSTANT_CFSTRINGS__ 1
#define __DBL_DECIMAL_DIG__ 17
#define __DBL_DENORM_MIN__ 4.9406564584124654e-324
#define __DBL_DIG__ 15
#define __DBL_EPSILON__ 2.2204460492503131e-16
#define __DBL_HAS_DENORM__ 1
#define __DBL_HAS_INFINITY__ 1
#define __DBL_HAS_QUIET_NAN__ 1
#define __DBL_MANT_DIG__ 53
#define __DBL_MAX_10_EXP__ 308
#define __DBL_MAX_EXP__ 1024
#define __DBL_MAX__ 1.7976931348623157e+308
#define __DBL_MIN_10_EXP__ (-307)
#define __DBL_MIN_EXP__ (-1021)
#define __DBL_MIN__ 2.2250738585072014e-308
#define __DECIMAL_DIG__ __LDBL_DECIMAL_DIG__
#define __DYNAMIC__ 1
#define __ENVIRONMENT_MAC_OS_X_VERSION_MIN_REQUIRED__ 101100
#define __FINITE_MATH_ONLY__ 0
#define __FLT_DECIMAL_DIG__ 9
#define __FLT_DENORM_MIN__ 1.40129846e-45F
#define __FLT_DIG__ 6
#define __FLT_EPSILON__ 1.19209290e-7F
#define __FLT_EVAL_METHOD__ 0
#define __FLT_HAS_DENORM__ 1
#define __FLT_HAS_INFINITY__ 1
#define __FLT_HAS_QUIET_NAN__ 1
#define __FLT_MANT_DIG__ 24
#define __FLT_MAX_10_EXP__ 38
#define __FLT_MAX_EXP__ 128
#define __FLT_MAX__ 3.40282347e+38F
#define __FLT_MIN_10_EXP__ (-37)
#define __FLT_MIN_EXP__ (-125)
#define __FLT_MIN__ 1.17549435e-38F
#define __FLT_RADIX__ 2
#define __FXSR__ 1
#define __GCC_ATOMIC_BOOL_LOCK_FREE 2
#define __GCC_ATOMIC_CHAR16_T_LOCK_FREE 2
#define __GCC_ATOMIC_CHAR32_T_LOCK_FREE 2
#define __GCC_ATOMIC_CHAR_LOCK_FREE 2
#define __GCC_ATOMIC_INT_LOCK_FREE 2
#define __GCC_ATOMIC_LLONG_LOCK_FREE 2
#define __GCC_ATOMIC_LONG_LOCK_FREE 2
#define __GCC_ATOMIC_POINTER_LOCK_FREE 2
#define __GCC_ATOMIC_SHORT_LOCK_FREE 2
#define __GCC_ATOMIC_TEST_AND_SET_TRUEVAL 1
#define __GCC_ATOMIC_WCHAR_T_LOCK_FREE 2
#define __GCC_HAVE_SYNC_COMPARE_AND_SWAP_1 1
#define __GCC_HAVE_SYNC_COMPARE_AND_SWAP_16 1
#define __GCC_HAVE_SYNC_COMPARE_AND_SWAP_2 1
#define __GCC_HAVE_SYNC_COMPARE_AND_SWAP_4 1
#define __GCC_HAVE_SYNC_COMPARE_AND_SWAP_8 1
#define __GNUC_MINOR__ 2
#define __GNUC_PATCHLEVEL__ 1
#define __GNUC_STDC_INLINE__ 1
#define __GNUC__ 4
#define __GXX_ABI_VERSION 1002
#define __GXX_RTTI 1
#define __INT16_C_SUFFIX__
#define __INT16_FMTd__ "hd"
#define __INT16_FMTi__ "hi"
#define __INT16_MAX__ 32767
#define __INT16_TYPE__ short
#define __INT32_C_SUFFIX__
#define __INT32_FMTd__ "d"
#define __INT32_FMTi__ "i"
#define __INT32_MAX__ 2147483647
#define __INT32_TYPE__ int
#define __INT64_C_SUFFIX__ LL
#define __INT64_FMTd__ "lld"
#define __INT64_FMTi__ "lli"
#define __INT64_MAX__ 9223372036854775807LL
#define __INT64_TYPE__ long long int
#define __INT8_C_SUFFIX__
#define __INT8_FMTd__ "hhd"
#define __INT8_FMTi__ "hhi"
#define __INT8_MAX__ 127
#define __INT8_TYPE__ signed char
#define __INTMAX_C_SUFFIX__ L
#define __INTMAX_FMTd__ "ld"
#define __INTMAX_FMTi__ "li"
#define __INTMAX_MAX__ 9223372036854775807L
#define __INTMAX_TYPE__ long int
#define __INTMAX_WIDTH__ 64
#define __INTPTR_FMTd__ "ld"
#define __INTPTR_FMTi__ "li"
#define __INTPTR_MAX__ 9223372036854775807L
#define __INTPTR_TYPE__ long int
#define __INTPTR_WIDTH__ 64
#define __INT_FAST16_FMTd__ "hd"
#define __INT_FAST16_FMTi__ "hi"
#define __INT_FAST16_MAX__ 32767
#define __INT_FAST16_TYPE__ short
#define __INT_FAST32_FMTd__ "d"
#define __INT_FAST32_FMTi__ "i"
#define __INT_FAST32_MAX__ 2147483647
#define __INT_FAST32_TYPE__ int
#define __INT_FAST64_FMTd__ "ld"
#define __INT_FAST64_FMTi__ "li"
#define __INT_FAST64_MAX__ 9223372036854775807L
#define __INT_FAST64_TYPE__ long int
#define __INT_FAST8_FMTd__ "hhd"
#define __INT_FAST8_FMTi__ "hhi"
#define __INT_FAST8_MAX__ 127
#define __INT_FAST8_TYPE__ signed char
#define __INT_LEAST16_FMTd__ "hd"
#define __INT_LEAST16_FMTi__ "hi"
#define __INT_LEAST16_MAX__ 32767
#define __INT_LEAST16_TYPE__ short
#define __INT_LEAST32_FMTd__ "d"
#define __INT_LEAST32_FMTi__ "i"
#define __INT_LEAST32_MAX__ 2147483647
#define __INT_LEAST32_TYPE__ int
#define __INT_LEAST64_FMTd__ "ld"
#define __INT_LEAST64_FMTi__ "li"
#define __INT_LEAST64_MAX__ 9223372036854775807L
#define __INT_LEAST64_TYPE__ long int
#define __INT_LEAST8_FMTd__ "hhd"
#define __INT_LEAST8_FMTi__ "hhi"
#define __INT_LEAST8_MAX__ 127
#define __INT_LEAST8_TYPE__ signed char
#define __INT_MAX__ 2147483647
#define __LDBL_DECIMAL_DIG__ 21
#define __LDBL_DENORM_MIN__ 3.64519953188247460253e-4951L
#define __LDBL_DIG__ 18
#define __LDBL_EPSILON__ 1.08420217248550443401e-19L
#define __LDBL_HAS_DENORM__ 1
#define __LDBL_HAS_INFINITY__ 1
#define __LDBL_HAS_QUIET_NAN__ 1
#define __LDBL_MANT_DIG__ 64
#define __LDBL_MAX_10_EXP__ 4932
#define __LDBL_MAX_EXP__ 16384
#define __LDBL_MAX__ 1.18973149535723176502e+4932L
#define __LDBL_MIN_10_EXP__ (-4931)
#define __LDBL_MIN_EXP__ (-16381)
#define __LDBL_MIN__ 3.36210314311209350626e-4932L
#define __LITTLE_ENDIAN__ 1
#define __LONG_LONG_MAX__ 9223372036854775807LL // long long 最大值
#define __LONG_MAX__ 9223372036854775807L  // long 最大值
#define __LP64__ 1 // This macro is defined in any Apple computer
#define __MACH__ 1  // both __APPLE__ and  __MACH__ indicate OSX.
#define __MMX__ 1
#define __NO_INLINE__ 1
#define __NO_MATH_INLINES 1
#define __ORDER_BIG_ENDIAN__ 4321
#define __ORDER_LITTLE_ENDIAN__ 1234
#define __ORDER_PDP_ENDIAN__ 3412
#define __PIC__ 2
#define __POINTER_WIDTH__ 64
#define __PRAGMA_REDEFINE_EXTNAME 1
#define __PTRDIFF_FMTd__ "ld"
#define __PTRDIFF_FMTi__ "li"
#define __PTRDIFF_MAX__ 9223372036854775807L
#define __PTRDIFF_TYPE__ long int
#define __PTRDIFF_WIDTH__ 64
#define __REGISTER_PREFIX__
#define __SCHAR_MAX__ 127
#define __SHRT_MAX__ 32767
#define __SIG_ATOMIC_MAX__ 2147483647
#define __SIG_ATOMIC_WIDTH__ 32
#define __SIZEOF_DOUBLE__ 8 // double 大小 下面依次类推
#define __SIZEOF_FLOAT__ 4
#define __SIZEOF_INT128__ 16
#define __SIZEOF_INT__ 4
#define __SIZEOF_LONG_DOUBLE__ 16
#define __SIZEOF_LONG_LONG__ 8
#define __SIZEOF_LONG__ 8
#define __SIZEOF_POINTER__ 8
#define __SIZEOF_PTRDIFF_T__ 8
#define __SIZEOF_SHORT__ 2
#define __SIZEOF_SIZE_T__ 8
#define __SIZEOF_WCHAR_T__ 4
#define __SIZEOF_WINT_T__ 4
#define __SIZE_FMTX__ "lX"
#define __SIZE_FMTo__ "lo"
#define __SIZE_FMTu__ "lu"
#define __SIZE_FMTx__ "lx"
#define __SIZE_MAX__ 18446744073709551615UL
#define __SIZE_TYPE__ long unsigned int
#define __SIZE_WIDTH__ 64
#define __SSE2_MATH__ 1
#define __SSE2__ 1
#define __SSE3__ 1
#define __SSE_MATH__ 1
#define __SSE__ 1
#define __SSP__ 1
#define __SSSE3__ 1
#define __STDC_HOSTED__ 1
#define __STDC_UTF_16__ 1
#define __STDC_UTF_32__ 1
#define __STDC_VERSION__ 201112L
#define __STDC__ 1
#define __UINT16_C_SUFFIX__
#define __UINT16_FMTX__ "hX"
#define __UINT16_FMTo__ "ho"
#define __UINT16_FMTu__ "hu"
#define __UINT16_FMTx__ "hx"
#define __UINT16_MAX__ 65535
#define __UINT16_TYPE__ unsigned short
#define __UINT32_C_SUFFIX__ U
#define __UINT32_FMTX__ "X"
#define __UINT32_FMTo__ "o"
#define __UINT32_FMTu__ "u"
#define __UINT32_FMTx__ "x"
#define __UINT32_MAX__ 4294967295U
#define __UINT32_TYPE__ unsigned int
#define __UINT64_C_SUFFIX__ ULL
#define __UINT64_FMTX__ "llX"
#define __UINT64_FMTo__ "llo"
#define __UINT64_FMTu__ "llu"
#define __UINT64_FMTx__ "llx"
#define __UINT64_MAX__ 18446744073709551615ULL
#define __UINT64_TYPE__ long long unsigned int
#define __UINT8_C_SUFFIX__
#define __UINT8_FMTX__ "hhX"
#define __UINT8_FMTo__ "hho"
#define __UINT8_FMTu__ "hhu"
#define __UINT8_FMTx__ "hhx"
#define __UINT8_MAX__ 255
#define __UINT8_TYPE__ unsigned char
#define __UINTMAX_C_SUFFIX__ UL
#define __UINTMAX_FMTX__ "lX"
#define __UINTMAX_FMTo__ "lo"
#define __UINTMAX_FMTu__ "lu"
#define __UINTMAX_FMTx__ "lx"
#define __UINTMAX_MAX__ 18446744073709551615UL
#define __UINTMAX_TYPE__ long unsigned int
#define __UINTMAX_WIDTH__ 64
#define __UINTPTR_FMTX__ "lX"
#define __UINTPTR_FMTo__ "lo"
#define __UINTPTR_FMTu__ "lu"
#define __UINTPTR_FMTx__ "lx"
#define __UINTPTR_MAX__ 18446744073709551615UL
#define __UINTPTR_TYPE__ long unsigned int
#define __UINTPTR_WIDTH__ 64
#define __UINT_FAST16_FMTX__ "hX"
#define __UINT_FAST16_FMTo__ "ho"
#define __UINT_FAST16_FMTu__ "hu"
#define __UINT_FAST16_FMTx__ "hx"
#define __UINT_FAST16_MAX__ 65535
#define __UINT_FAST16_TYPE__ unsigned short
#define __UINT_FAST32_FMTX__ "X"
#define __UINT_FAST32_FMTo__ "o"
#define __UINT_FAST32_FMTu__ "u"
#define __UINT_FAST32_FMTx__ "x"
#define __UINT_FAST32_MAX__ 4294967295U
#define __UINT_FAST32_TYPE__ unsigned int
#define __UINT_FAST64_FMTX__ "lX"
#define __UINT_FAST64_FMTo__ "lo"
#define __UINT_FAST64_FMTu__ "lu"
#define __UINT_FAST64_FMTx__ "lx"
#define __UINT_FAST64_MAX__ 18446744073709551615UL
#define __UINT_FAST64_TYPE__ long unsigned int
#define __UINT_FAST8_FMTX__ "hhX"
#define __UINT_FAST8_FMTo__ "hho"
#define __UINT_FAST8_FMTu__ "hhu"
#define __UINT_FAST8_FMTx__ "hhx"
#define __UINT_FAST8_MAX__ 255
#define __UINT_FAST8_TYPE__ unsigned char
#define __UINT_LEAST16_FMTX__ "hX"
#define __UINT_LEAST16_FMTo__ "ho"
#define __UINT_LEAST16_FMTu__ "hu"
#define __UINT_LEAST16_FMTx__ "hx"
#define __UINT_LEAST16_MAX__ 65535
#define __UINT_LEAST16_TYPE__ unsigned short
#define __UINT_LEAST32_FMTX__ "X"
#define __UINT_LEAST32_FMTo__ "o"
#define __UINT_LEAST32_FMTu__ "u"
#define __UINT_LEAST32_FMTx__ "x"
#define __UINT_LEAST32_MAX__ 4294967295U
#define __UINT_LEAST32_TYPE__ unsigned int
#define __UINT_LEAST64_FMTX__ "lX"
#define __UINT_LEAST64_FMTo__ "lo"
#define __UINT_LEAST64_FMTu__ "lu"
#define __UINT_LEAST64_FMTx__ "lx"
#define __UINT_LEAST64_MAX__ 18446744073709551615UL
#define __UINT_LEAST64_TYPE__ long unsigned int
#define __UINT_LEAST8_FMTX__ "hhX"
#define __UINT_LEAST8_FMTo__ "hho"
#define __UINT_LEAST8_FMTu__ "hhu"
#define __UINT_LEAST8_FMTx__ "hhx"
#define __UINT_LEAST8_MAX__ 255
#define __UINT_LEAST8_TYPE__ unsigned char
#define __USER_LABEL_PREFIX__ _
#define __VERSION__ "4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.42.1)"
#define __WCHAR_MAX__ 2147483647
#define __WCHAR_TYPE__ int
#define __WCHAR_WIDTH__ 32
#define __WINT_TYPE__ int
#define __WINT_WIDTH__ 32
#define __amd64 1
#define __amd64__ 1
#define __apple_build_version__ 8000042
#define __block __attribute__((__blocks__(byref)))
#define __clang__ 1  //Defined when compiling with Clang
#define __clang_major__ 8 //the major marketing version number of Clang
#define __clang_minor__ 0
#define __clang_patchlevel__ 0
#define __clang_version__ "8.0.0 (clang-800.0.42.1)"
#define __core2 1
#define __core2__ 1
#define __llvm__ 1
#define __nonnull _Nonnull
#define __null_unspecified _Null_unspecified
#define __nullable _Nullable
#define __pic__ 2
#define __strong
#define __tune_core2__ 1
#define __unsafe_unretained
#define __weak __attribute__((objc_gc(weak)))
#define __x86_64 1
#define __x86_64__ 1
```




