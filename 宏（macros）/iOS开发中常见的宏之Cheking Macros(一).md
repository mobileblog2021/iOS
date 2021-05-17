



我们在Objective-C中判断某一方法在当前平台(iOS or Mac)和当前系统版本（iOS7.0 or iOS8.0)是否可用,一般是这么做的

```objective-c
// in UIViewController
if ([self respondsToSelector:@selector(edgesForExtendedLayout)]) {
      self.edgesForExtendedLayout = UIRectEdgeNone;
  }
```

那么像似的我们怎么来判断当前的编译器（Clang）,是否支持某一内置函数、特性、属性呢？

> Clang提供了很多非常有用的语言扩展（Language extensions），为了确保我们能正确使用这些扩展功能，Clang提供了几种内置的类函数宏（相当于我们常用的`respondsToSelector`），这些宏可以帮助我们检测这些扩展的有效性，而不用根据编译器版本来检测可用性这种比较恶心的方式。

本篇文章主要来介绍一些这些特性检查宏（Feature Checking Macros）

### `__has_builtin`

此函数类型的宏传递一个函数名作为参数来判断该函数是否为内置函数。

```c
int popcount(int x)
{
#if __has_builtin(__builtin_popcount)
    return __builtin_popcount(x); // 返回二进制中`1`的个数 (内置函数性能更好)
#else
    int count = 0;
    while(x){
        x &= x - 1; // 逐步将x最右边的1清除
        count++;
    }
    return count;
#endif
}
```

### `__has_feature` & `__has_extension`

此两个函数类型的宏传递一个特性的名字作为参数来判断是否被支持。

`__has_feature`是判断是否符合Clang和当前语言的标准 ;

`__has_extension`是判断是否符合Clang和当前语言的标准（不管是该语言的扩展还是标准）;

```objective-c
// 是否支持instancetype
#if __has_feature(objc_instancetype)
    #define MB_INSTANCETYPE instancetype
#else
    #define MB_INSTANCETYPE id
#endif

// 是否支持blocks
#if __has_extension(blocks)
     NSLog(@"support blocks");
#endif
```

出于向后兼容的考虑，`__has_feature`也可以用来检查非标准语法特性，如：不是以`c_`、`cxx_`、`objc_`等为前缀的特性。所以用`__has_feature(blocks)`来检查是否支持block也是可以的。如果设置`-pedantic-errors`选项，`__has_extension`和`__has_feature`作用就是一样的。

### `__has_attribute`

此宏传递一个GNU-style属性名称作为参数用于检查是否被支持。

```c
#if __has_attribute(always_inline)
#define ALWAYS_INLINE __attribute__((always_inline))
#else
#define ALWAYS_INLINE
#endif

// 强制编译器内联此函数，即使相关的优化选项被关闭
static inline ALWAYS_INLINE int fn(const char *s)
{
  return (!s || (*s == '\0'));
}
```

传入的属性名也可以采用前后加`__`（下划线）的命名方式来防止命名冲突，所以这里`__always_inline__`和`always_inline`是等价的。

### `__is_identifier`

此宏传递一个保留关键字(reserved word)或者一个标识符(regular identifier)作为参数用于检查是否被支持。如果只是一个标识符而不是保留关键字 ，则返回1，也可以用来检测用户自定义的方法或变量。

```c
#ifdef __is_identifier          
  #if __is_identifier(__wchar_t)
    typedef wchar_t __wchar_t;
  #endif
#endif
```

### 常用特性检查

```objective-c
// 检查是否支持blocks
__has_extension(blocks)  
  
// 检查是否支持instancetye上下文关键词
__has_feature(objc_instancetype)
  
// 检查是否支持arc
__has_feature(objc_arc) 
  
// 同时检查是否支持__weak指针
__has_feature(objc_arc_weak)
 
// 检查是否支持固定基础类型的枚举
__has_feature(objc_fixed_enum)
  
// 检查是否支持对象字面值 示例：NSArray *array = @[];
__has_feature(objc_array_literals) 
__has_feature(objc_dictionary_literals) 
  
// 检查是否支持对象下标（OC的对象指针现在可以像C一样做下标操作）示例：dictionary[key]
__has_feature(objc_subscripting)
  
// 检查是否支持属性的自动合成（不使用@dynamic的情况下，自动生成存取方法）
__has_feature(objc_default_synthesize_properties) 
  
// 是否支持the new nullability annotations
__has_feature(nullability)

// 示例
#if __has_feature(nullability)
   #define __ASSUME_NONNULL_BEGIN      NS_ASSUME_NONNULL_BEGIN
   #define __ASSUME_NONNULL_END        NS_ASSUME_NONNULL_END
   #define __NULLABLE                  nullable
#else
   #define __ASSUME_NONNULL_BEGIN
   #define __ASSUME_NONNULL_END
   #define __NULLABLE
#endif
  
// 是否支持objc泛型
__has_feature(objc_generics)
 
// 示例
#if __has_feature(objc_generics)
    #define __GENERICS(class, ...)      class<__VA_ARGS__>
    #define __GENERICS_TYPE(type)       type
#else
    #define __GENERICS(class, ...)      class
    #define __GENERICS_TYPE(type)       id
#endif
  
```



在Xcode8中，我们之前用的好好的插件不能用了（去掉签名的Xcode也不稳定），但是iOS10，引入了User UserNotificationsUI.framework，允许我们定制本地或远程通知出现在设备上时的外观，更新代码后，我们发现在Xcode7中编译错误。

那我们通过代码来兼容不同的Xcode的版本呢？

方法一 ：SDK 'iOS 10.0' (Xcode 8) 给出了更多的版本号码甚至是特征版本(小版本)

```c
#define NSFoundationVersionNumber_iOS_9_0 1240.1
#define NSFoundationVersionNumber_iOS_9_1 1241.14
#define NSFoundationVersionNumber_iOS_9_2 1242.12
#define NSFoundationVersionNumber_iOS_9_3 1242.12
#define NSFoundationVersionNumber_iOS_9_4 1280.25
#define NSFoundationVersionNumber_iOS_9_x_Max 1299
```

所以我们可以这样（这样依赖于一些宏的定义）

```objective-c
#ifdef NSFoundationVersionNumber_iOS9x_Max
   NSLog(@"Xcode8");
#else
   NSLog(@"Xcode7 or below");
#endif
```

方法二：一种更通用的方法

```objective-c
#if __has_include(<UserNotifications/UserNotifications.h>)
    NSLog(@"Xcode8");
    // do sth use UserNotifications/UserNotifications.h
#else
    NSLog(@"Xcode7 or below");
    // do nothing
#endif
```



### `__has_include`  &  ` __has_include_next`

此宏可以检测一个 #include or #include_next 文件是否存在

```
#if defined(__has_include)
  #if __has_include("myinclude.h")
     #include "myinclude.h"
  #endif
#endif
```

而#import实质上做的事情和#include是一样的，只不过OC为了避免重复引用可能带来的编译错误。所以我们可以用__has_include 来检测 #import 的文件是否存在



### `__has_warning`

此宏传递一个字符串，用于检测这个字符串是否是一个合法的警告选项

```c
#if __has_warning("-Wformat")
  // do sth
#endif
```



###  `#if`  &  `#ifdef       `  &  `#if defined(NAME)` 

`#if` 后面接的是表达式，如果成立则把对应的代码编译进去

```c
#if (MAX==10)||(MAX==20) 
  // do sth 
#endif
```

`ifdef`和  `#if defined(NAME)` 检测 宏有没有定义，两者的区别是`#ifdef`只能接一个条件

```c
#if defined(WIN32) && !defined(UNIX)
   // Do windows stuff
#elif defined(UNIX) && !defined(WIN32)
   // Do linux stuff
#else
   // Error, both can't be defined or undefined same time
#endif
```









