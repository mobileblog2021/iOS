### 这个宏是怎么生效的？

```objective-c
// DEBUG模式下
// @weakify(self) 展开后 
@autoreleasepool {} __attribute__((objc_ownership(weak))) __typeof__(self) self_weak_ = (self);
  
// @strongify(self) 展开后
@autoreleasepool {}
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wshadow"
 __attribute__((objc_ownership(strong))) __typeof__(self) self = self_weak_;
#pragma clang diagnostic pop
```

我们过后再研究它是怎么展开成这个样子的，先来看看上面的代码是怎么打破循环引用的。

我们先来简化一下代码

```c
// 暂时去掉其他辅助的代码
__attribute__((objc_ownership(weak))) __typeof__(self) self_weak_ = (self);

__attribute__((objc_ownership(strong))) __typeof__(self) self = self_weak_;
```

```shell
# 在ARC下，修改对象所有权（ownership modifiers）的关键字，通过以下内置宏定义
$ clang -E -dM -x objective-c -fobjc-arc /dev/null | egrep  "weak|strong|unsafe"
#define __strong __attribute__((objc_ownership(strong)))
#define __unsafe_unretained __attribute__((objc_ownership(none)))
#define __weak __attribute__((objc_ownership(weak)))
```

```c
// 进一步简化为如下代码
__weak __typeof__(self) self_weak_ = (self);

__strong __typeof__(self) self = self_weak_;


// 两个参数的情况  @weakify(self,obj)
 __attribute__((objc_ownership(weak))) __typeof__(self) self_weak_ = (self);
__attribute__((objc_ownership(weak))) __typeof__(_obj) _obj_weak_ = (_obj);

 __attribute__((objc_ownership(strong))) __typeof__(self) self = self_weak_;
 __attribute__((objc_ownership(strong))) __typeof__(_obj) _obj = _obj_weak_;
```

现在看是不是很眼熟了，但是还是有一点不一样，这里又定义了一个局部变量叫self,我们知道一个同名的局部变量会覆盖（shadow）对应的成员变量，这也是通过`-Wshadow`来消除警告的原因。

```objective-c
// xcode8和xcode7中测试没有警告  
// 当前定义了一个和当前类同样生命周期的强引用self(成员变量),这样block不会保持self的引用。
__weak __typeof__(self) self_weak_ = (self);
BOOL (^testBlock)(id) = ^ BOOL (id obj){
    __strong __typeof__(self) __unused self = self_weak_;
    //这里定义了一个作用域在block内的强引用self(局部变量),并会一直存货，知道block结束，保证中途不会被置为nil,使逻辑被中途打断，或crash
    return YES;
};

self.testBlock=testBlock;
```

self引用着testBlock,testBlock引用着一个局部变量self，这样循环引用就被打破了。

小扩展：`__typeof__` 是获得对应变量的类型,和`__typeof()`都是c语言的编译时扩展，`typeof,` `__typeof` and` __typeof__ `在Objective-C中代表同样的意思。

```objective-c
  NSString *test=@"a";
  typeof(test) a; // a的数据类型是NSString 等价于 NSString *a;
```

### 这个宏是怎么展开的？

到此我们应该明白了@weakify(...) & @strongify(…)展开后的效果。那么这个宏是怎么展开的呢？

 

```c
#define weakify(...) rac_keywordify metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)
```

```objective-c
// 只为模仿系统系统的关键词前加了一个@ ， 下面的两种做法都不完美，作者做了一个权衡
#if DEBUG
#define rac_keywordify autoreleasepool {}  // 空的也不会被编译器优化去掉
#else
#define rac_keywordify try {} @catch (...) {} // 在有返回值的block中会抑制编译器对没有return的错误提示
#endif
```

```c
// 这样weakify简化为
#define weakify(...)  metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)

#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(__VA_ARGS__))(MACRO, SEP, CONTEXT, __VA_ARGS__)
```

metamacro_foreach_cxt这个宏语法上看着有点让人费解，但是我们只是去做字符串替换，就会渐渐明白了；这里为了不打断宏的展开，我们先只说一下metamacro_argcount(`__VA_ARGS__`)的作用，即获得可变参数的个数，后面我们再详细研究。

```c
#define metamacro_concat(A, B) A ## B // 即将两个参数名拼接在一起

// 假设我么这里的可变的参数的个数是2 ，即 metamacro_argcount(__VA_ARGS__) 等价于2
// 所以metamacro_foreach_cxt 简化为一下形式
// MACRO :rac_weakify_
// SEP: 空
// CONTEXT:__weak
// 这个宏类似一共21个 0-20 所以weakify(…) 最多支持20个可变参数
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...)  \
metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, __VA_ARGS__)  

// 这样weakify简化为
#define weakify(...) metamacro_foreach_cxt2(rac_weakify_, , __weak, __VA_ARGS__)
```

```c
// 因为我们已经知道可变参数的长度，就可以把可变参数，改变为固定参数
// _0就是weakify(...)jw第一个参数名  _1就是weakify(...)中第二个参数名 
#define metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, _0, _1) \
    metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) \
    SEP \
    MACRO(1, CONTEXT, _1)
    
// 替换参数
#define weakify(...)   \
    metamacro_foreach_cxt1(rac_weakify_, , __weak, _0) \
     \
    rac_weakify_(1, __weak, _1)

#define rac_weakify_(INDEX, CONTEXT, VAR) \
    CONTEXT __typeof__(VAR) metamacro_concat(VAR, _weak_) = (VAR);

// 替换rac_weakify_   
#define weakify(...)   \
    metamacro_foreach_cxt1(rac_weakify_, , __weak, _0) \
     \
    __weak __typeof__(VAR) _1_weak_= (_1);

// 同理展开metamacro_foreach_cxt1
#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)

// 最终 weakify(_0,_1) 
#define weakify(...)   \
    __weak __typeof__(_0) _0_weak_= (_0); \
     \
    __weak __typeof__(_1) _1_weak_= (_1);

// 示例
#define weakify(self,obj)   \
    __weak __typeof__(self) self_weak_= (self); \
     \
    __weak __typeof__(obj) obj_weak_= (obj);
```

### 怎么获得可变参数宏的参数个数？

```c
#define metamacro_argcount(...) \
        metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
     
// 举个例子， 走一下看看
metamacro_at(arg0,arg1)
// 展开后
metamacro_at(20, arg0,arg1, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)

// 返回第N个可变参数(从0开始)，这需要至少提供N+1个参数，0<=N<=5
#define metamacro_at(N, ...) metamacro_concat(metamacro_at, N)(__VA_ARGS__)  
  
// 继续展开
metamacro_at20(arg0,arg1, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
 
// 这个宏类似的有一共21个。
// 这里 前20是固定参数，剩下的是可变参数
#define metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(__VA_ARGS__)
 
// 继续展开 
// arg0,arg1, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1
// 前20个是固定参数，后两个是可变参数,所以只剩下2，1
metamacro_head(2, 1)
 
// 获得第一个参数的值
#define metamacro_head(...) metamacro_head_(__VA_ARGS__, 0)  
#define metamacro_head_(FIRST, ...) FIRST
  
// 继续展开 
metamacro_head_(2，1, 0)
2  
```

可见作者通过控制`__VA_ARGS__`的位置，来获得可变参数的个数



**另外@unsafeify(…)和@strongify(...)配合使用，用于一些不支持__weak 的class **




