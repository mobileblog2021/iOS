我们先来提几个问题

1、怎么在别人错误的使用你的API时，给予警告？

2、怎么避免在具体的情况下，消除警告？

3、在编译器层面，怎样尽可能的优化你的代码？

4、怎么在别人使用你的API时，给出更多信息？

为解决这些问题，我们来一起研究一个编译器指令`__attribute__`

`__attribute__`可以设置函数属性（Function Attribute ）、变量属性（Variable Attribute ）和类型属性（Type Attribute ）。

` __attribute__` 指令在C,C++,和Objective-C中用来修饰代码声明（函数，变量，参数，方法，类),这样就可以帮助编译器优化代码，给使用者给出警告。也可以说，`__attribute__`提供上下文，这有利于编译器的编译，另一个开发者和将来的你，这样也可以使你写出的代码性能尽可能的好。

当然提供错误的上下文，还不如不提供，由此引起的错误通常很难被发现，但这些挑战对我们来说。又算得了什么呢！

```c
// __attribute__语法：以逗号作为分割的一组属性
__attribute__ ((attribute-list))
```

###  `__attribute__((format))`

该__attribute__属性可以给被声明的函数加上类似printf或者scanf的特征，它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。

```c
// archetype: 哪种风格(__NSString__,printf,scanf,strftime,strfmon) 
// string-index:传入函数的第几个参数是格式化字符串(从1开始)
// first-to-check:第一个可变参数所在的索引
format (archetype, string-index, first-to-check)
```

```objective-c
FOUNDATION_EXPORT void NSLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2) NS_NO_TAIL_CALL;

// NSLog 涉及的两个属性
#if !defined(NS_FORMAT_FUNCTION)
    #if (__GNUC__*10+__GNUC_MINOR__ >= 42) && (TARGET_OS_MAC || TARGET_OS_EMBEDDED)
	#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))
    #else
	#define NS_FORMAT_FUNCTION(F,A)
    #endif
#endif

#if __has_attribute(not_tail_called)
   #define NS_NO_TAIL_CALL __attribute__((not_tail_called))
#else
   #define NS_NO_TAIL_CALL
#endif
```

例如：加上`NS_FORMAT_FUNCTION(1,2)`该属性后`NSLog(@"测试：%s",3);`,这样参数和格式化字符串不匹配的情况。编译器会给出警告。



### `__attribute__((not_tail_called))`

该__attribute__属性，在静态绑定调用（statically bound call）中禁用尾调用优化（tail-call optimization）,但在间接调用，虚方法，Objective-c 函数和标记为always_inline的方法，不能标记为not_tail_called

```c
int __attribute__((not_tail_called)) foo1(int);

int foo2(int a) {
  return foo1(a); // 在直接调用时，不会进行尾调用优化.
}
```

猜测NSLog 关闭尾调用优化，可能是为了保留函数调用栈，因为尾调用优化可以使栈帧消失。

### `__attribute__((cleanup(...)))`

该attribute修饰一个变量，**在它的作用域结束时可以自动执行一个指定的方法**

```objective-c
// 指定一个cleanup方法，注意入参是所修饰变量的地址，类型要一样
// 对于指向objc对象的指针(id *)，如果不强制声明__strong默认是__autoreleasing，造成类型不匹配
// 也可以自定义Class,基本类型，block
static void stringCleanUp(__strong NSString **string) {
    NSLog(@"%@", *string);
}
// 在某个方法中：
{
    __strong NSString *string __attribute__((cleanup(stringCleanUp))) = @"sunnyxx";
} // 当运行到这个作用域结束时，自动调用stringCleanUp
```

所谓作用域结束，包括大括号结束、return、goto、break、exception等各种情况。

假如一个作用域内有若干个cleanup的变量，他们的调用顺序是`先入后出`的栈式顺序；
而且，cleanup是先于这个对象的`dealloc`调用的。



### `__attribute__((constructor/destructor))`

若函数被设定为constructor属性，则该函数会在main（）函数执行之前被自动的执行;

若函数被设定为destructor属性，则该 函数会在main（）函数执行之后或者exit（）被调用后被自动的执行。

```c
static __attribute__((constructor)) void before() {
    printf("Hello");
}

static __attribute__((destructor)) void after() {
    printf("World!\n");
}

int main(int argc, const char * argv[]) {
    // insert code here...
    printf("_main_");
    return 0;
}
```

程序输出结果是：

```
Hello_main_World!
```

如果多个函数使用了这个属性，可以为它们设置优先级来决定执行的顺序：

```c
__attribute__((constructor(PRIORITY))) // 越小越优先执行 (0<PRIORITY<=100是内部保留的)
__attribute__((destructor(PRIORITY)))  // 越大越优先执行
```

### `__attribute__((deprecated(message)))`

```objective-c
- (NSData *)md5Digest XMPP_DEPRECATED("Use -xmpp_md5Digest instead");
```

### `__attribute__((warn_unused_result))`

添加该属性后，调用时只是简单地调用,却并没有用一个变量接收,系统就会提示`Ignoring return value of function declared with warn_unused_result attribute`

```
- (BOOL)test:(NSInteger)num __attribute__((warn_unused_result)){
    return num == 1?YES:NO;
}
```

### `__attribute__((noreturn))`

该属性通知编译器函数从不返回值，当遇到类似函数需要返回值而却不可能运行到返回值处就已经退出来的情况，该属性可以避免出现错误信息。C库函数中的abort（）和exit（）的声明格式就采用了这种格式。

```c
extern void myexit() __attribute__((noreturn));

int test(int n)
{
    if ( n > 0 ){
        myexit();
        /* 程序不可能到达这里*/
    }
    else{
        return 0;
    }
}
```

如果不加`__attribute__((noreturn))`,因为你定义了一个有返回值的函数test却有可能没有返回值，程序当然不知道怎么办了，则给出警告`control reaches end of non-void function`。

### `__attribute__((const))` & `__attribute__((pure))`

`__attribute__((const))`属性只能用于带有数值类型参参数的函数上。当重复调用带有数值类型参参数的函数时，由于返回值是相同的，所以此时编译器可以进行优化处理，除第一次需要运算外，其它只需要返回第一次的结果就可以了，进而可以提高效率。该属性主要适用于没有副作用的一些函数，并且返回值仅仅依赖输入的参数

```c
extern int square(int n) __attribute__((const));
int test(int n)
{     
     for (i = 0; i < 100; i++ )
     {
           total += square(5) + i;
     }
}
```

通过添加`__attribute__((const))`声明，编译器只调用了函数一次，以后只是直接得到了相同的一个返回值`__attribute__((pure))`，比const条件稍微宽松一些，标明了函数除了依赖其参数外，还可以依赖一些global/static的变量.

推荐所有单例类的sharedInstance方法都使用const注解。此外，如果一个类的类方法经常被调用而且是const或者pure，那么不妨把它用纯C函数实现，会有不错的性能提升。

### `__attribute__((__no_instrument_function__))`

GCC`-finstrument-functions`参数可以使程序在编译时，在函数的入口和出口处生成instrumentation调用。恰好在函数入口之后并恰好在函数出口之前，将使用当前函数的地址和调用地址来调用下面的 profiling 函数。（在一些平台上，__builtin_return_address不能在超过当前函数范围之外正常工作，所以调用地址信息可能对profiling函数是无效的。）

```c
#define DUMP(func, call) printf("%s: func = %p, called by = %p/n", __FUNCTION__, func, call)

// this_func:当前函数的起始地址 ，可在符号表中找到;
// call_site:调用处地址
void __attribute__((__no_instrument_function__))
__cyg_profile_func_enter(void *this_func, void *call_site)
{
    DUMP(this_func, call_site);
}
void __attribute__((__no_instrument_function__))
__cyg_profile_func_exit(void *this_func, void *call_site)
{
    DUMP(this_func, call_site);
}


int main(int argc, const char * argv[]) {
    printf("_main_\n");
    return 0;
}
```

```shell
  gcc -finstrument-functions  -c main.c
  gcc -finstrument-functions -finstrument-functions main.o -o main
  ./main
  __cyg_profile_func_enter: func = 0x1023dded0, called by = 0x7fff9dcb55ad/n_main_
  __cyg_profile_func_exit: func = 0x1023dded0, called by = 0x7fff9dcb55ad/n%
```

可对函数指定no_instrument_function属性，在这种情况下不会进行instrumentation操作。例如，可以在以下情况下使用no_instrument_function属性：上面列出的profiling函数、高优先级的中断例程以及任何不能保证profiling正常调用的函数。



### `__attribute__((always_inline))`

此函数属性指示必须内联函数。

编译器将尝试内联函数，而不考虑函数的特性。但是，如果这样做导致出现问题，编译器将不内联函数。例如，递归函数仅内联到本身一次。

```
static __attribute__((always_inline))  int max(int x, int y)
{
    return x > y ? x : y; // always inline if possible
}
```

内联函数的使用场景：1、使用inline 代替 #define

​                                      2、函数非常小，且调用非常频繁

### `__attribute__((deprecated))`

该属性 用来标识一个方法，变量，类型预期在将来的版本中被移除。

```objective-c
// 有两个可选的 string 类型的参数
// arg0:在警告时展示
// arg1:使编译器提供一个可替换的的方案
void f(void) __attribute__((deprecated("message", "replacement")));

__attribute__((deprecated))
@interface DeprecatedClass : NSObject { ... }
...
@end
```
苹果也提供了`AvailabilityMacros.h`,包含DEPRECATED_ATTRIBUTE和DEPRECATED_MSG_ATTRIBUTE(msg)

```objective-c
#define DEPRECATED_ATTRIBUTE        __attribute__((deprecated)) // 只在macOS/iOS下
#define DEPRECATED_MSG_ATTRIBUTE(s) __attribute__((deprecated(s)))
```

### `__attribute__((availability))`

该属性可以指示一个声明（方法，成员变量，类）在操作系统版本层面上的生命周期。

```c
// arg0: 平台的名称(ios,macos,tvos,watchos)
// introduced:第一个引入该方法的系统版本
// deprecated:第一个不宜用该方法的系统版本,意味着不应该再使用该API，warning
// obsoleted :第一个废弃该方法的版本，意味着API不再可用,error
// unavailable : 该生声明在当前平台从不可用
// message=string-literal :在展示警告或错误时,提供给用户额外的信息
// replacement=string-literal :展示可替代方案信息
void f(void) __attribute__((availability(macos,introduced=10.4,deprecated=10.6,obsoleted=10.7)));
```

示例：

```c
// 以下都是在iOS平台下

// 1、NS_DEPRECATED：是在描述这个声明被启用/停用的版本
#define NS_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, ...) CF_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, __VA_ARGS__)
#define CF_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, ...) __attribute__((availability(ios,introduced=_iosIntro,deprecated=_iosDep,message="" __VA_ARGS__)))

NS_DEPRECATED(NA,NA,5_0,6_0,"此方法已被某方法替代")
// 展开后 表示在iOS5以后可用 ，iOS6以后弃用
__attribute__((availability(ios,introduced=5_0,deprecated=6_0,message="" "此方法已被某方法替代")));


// 2、NS_AVAILABLE: 是在描述这个声明是在哪个版本开始启用
#define NS_AVAILABLE(_mac, _ios) CF_AVAILABLE(_mac, _ios)
#define CF_AVAILABLE(_mac, _ios) __attribute__((availability(ios,introduced=_ios)))

NS_AVAILABLE(NA, 9_0);
// 展开后 表示在iOS9以后可用
__attribute__((availability(ios,introduced=9_0)));
```

### `__attribute__((nonnull))`

此函数属性指定不假定为空指针的函数参数。这将使编译器在遇到此类参数时生成警告。

```c
// 如果未指定参数索引列表，则所有指针参数都被标记为非零
__attribute__((nonnull(arg-index, ...)))

void * my_memcpy (void *dest, const void *src, size_t len) __attribute__((nonnull (1, 2)));
```

### `__attribute__((unused))`

`unused` 函数属性禁止编译器在未引用该函数时生成警告。

__unused 关键字起到同样的作用，在代理方法，协议方法中很常用。

```objective-c
+ (void) __attribute__((noreturn)) networkRequestThreadEntryPoint:(id)__unused object {
    do {
        @autoreleasepool {
            [[NSRunLoop currentRunLoop] run];
        }
    } while (YES);
}

static int Function_Attributes_unused_0(int b) __attribute__ ((unused));
```

### `__attribute__(visibility)`

此函数属性影响 ELF 符号的可见性

```c
// default:假定的符号可见性可通过其他选项进行更改。缺省可见性将覆盖此类更改。缺省可见性与外部链接对应。
// hidden:该符号不存放在动态符号表中，因此，其他可执行文件或共享库都无法直接引用它。使用函数指针可进行间接引用。
// internal:除非由 特定于处理器的应用二进制接口 (psABI) 指定，否则，内部可见性意味着不允许从另一模块调用该函数。
// protected:该符号存放在动态符号表中，但定义模块内的引用将与局部符号绑定。也就是说，另一模块无法覆盖该符号。
__attribute__((visibility("visibility_type")))
  
// 示例
#define NS_CLASS_AVAILABLE(_mac, _ios) __attribute__((visibility("default"))) NS_AVAILABLE(_mac, _ios)
```

从动态共享库中尽可能少地输出符号是一个好的实践经验。输出一个受限制的符号会提高程序的模块性，并隐藏实现的细节。在库中减少符号的数目还可以减少库的内存印迹，减少动态连接器的工作量。动态连接器装载和识别的符号越少，程序启动和运行的速度就越快。

### `__attribute__((aligned))`

`aligned` 类型属性指定类型的最低对齐要求。

### `__attribute((packed))`

`packed` 类型属性指定类型必须具有最小的可能对齐要求。

### `__attribute__((overloadable))`

在C中不存在函数重载，不过LLVM提供了额外的扩展，实现了C函数的重载。要是用这个特性也十分简单，只需要在声明函数时加上__attribute__((overloadable))即可

```
#include <stdio.h>
 
__attribute__((overloadable)) void foo(int a){
    printf("%d\n",a);
}

__attribute__((overloadable)) void foo(double a){
    printf("%lf\n",a);
}
```

那么LLVM是怎么处理这个overloadable属性的呢？我们编译一下看看生成的函数符号。

```shell
$ clang -dynamiclib overloadable.c -o overloadable.dylib
$ nm overloadable.dylib

# 输出结果
 0000000000000f40 T __Z3food
 0000000000000f10 T __Z3fooi
                  U _printf
                  U dyld_stub_binder
```

我们可以看到，LLVM生成的函数符号是`__Z3food`和`__Z3fooi`,实际上这和C++的函数符号生成的方式是一样的。

### `__attribute__((sentinel(...))`

该属性表示当前方法或函数需要个nil(NULL)参数，作为分割-也叫哨兵（sentinel），多用于可变参数

```objective-c

#define NS_REQUIRES_NIL_TERMINATION __attribute__((sentinel(0,1)))

@interface NSArray
- (instancetype)arrayWithObjects:... NS_REQUIRES_NIL_TERMINATION;
@end
```

### `__attribute__((objc_requires_super))`

该属性表示在重写该方法时，必须调用super方法，否则给出警告

```
#define NS_REQUIRES_SUPER __attribute__((objc_requires_super))

@interface MyBaseClass : NSObject
- (void)handleStateTransition NS_REQUIRES_SUPER;
@end
```
### `__attribute__((objc_precise_lifetime)) `

该属性表示当前变量在作用域内有效，因为在release模式，ARC允许对象在最后一次使用后释放，没必要等到作用域走完，

```objective-c
#define NS_VALID_UNTIL_END_OF_SCOPE __attribute__((objc_precise_lifetime))

- (void)foo
{
    NS_VALID_UNTIL_END_OF_SCOPE MyObject *obj = [[MyObject alloc] init];
    NSValue *value = [NSValue valueWithPointer:obj];
    // do stuff
    MyObject *objAgain = [value pointerValue];
    NSLog(@"%@", objAgain);
}
```

在ARC下，如果没有添加NS_VALID_UNTIL_END_OF_SCOPE，当obj对象被用来创建NSValue后，编译器就不可能知道NSValue中的obj的使用情况，ARC会释放obj,NSLog会因EXEC_BAD_ACCESS崩溃。

### `__attribute__((ns_returns_retained))`  等 

在Objective-C中，方法通常是遵循[Cocoa Memory Management](http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html)规范来对参数和返回值进行内存管理的。但是会有特许的情况，所以Clang提供了属性来标识对象引用情况。

ns_returns_retained：标记为此属性的返回对象引用计数+1了；

ns_returns_not_retained: 标记为此属性的返回对象引用计数没有改变；

ns_returns_autorelased: 标记为该属性的返回对象引用计数+0，但是会在下一个自动释放池中被释放掉；

ns_consumed ：标记在方法中会被+1的参数；

ns_consumes_self：在Objective-C方法中，标记在该方法中self对象的引用计数+1；

### `__attribute__((objc_method_family(none)))`

在Objective-C中方法的命名通常代表着方法类型，如以`init`开头的方法会被编译器默认为是初始化方法，在编译的时候会将该方法当成初始化方法对待。但是有时候我们并不想以编译器默认的方式给方法取名，或者编译器默认的方法类型与我们自己想表示的有出入。我们就可以使用`__attribute__((objc_method_family(X)))`来明确说明该方法的类型，其中X取值为：`none`, `alloc`, `copy`, `init`, `mutableCopy`, `new`。如：

```objective-c
// 该方法在编译时就不会再被编译器当做初始化方法了
- (NSString *)initMyStringValue __attribute__((objc_method_family(none)));
```

