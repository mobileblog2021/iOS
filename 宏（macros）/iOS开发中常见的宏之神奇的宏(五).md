### @onExit

@onExit 定义了一些代码，会在作用域结束的时候执行，代码必须先来花括号里面，并以分号结束，不管作用域是怎么结束的，这些代码都会执行

作用域结束的方式：花括号结束,goto,return,break,continue,exception

      这些这些代码，会放在一个block中，在作用域结束后执行。注意关于于内存管理的问题和内存分配方面的限制。因为这些代码在block中调用，可以使用return提起退出cleanup block

      在同一作用域内，多个@onExit声明,调用顺序是先入后出的栈式顺序。这在一些代码需要配对使用的时候非常有用,因为这保证了在结束的时候，以逆序的方式执行。

      注意：这个声明，不能在一个没有使用花括号的作用域内使用（例如 if 单行，没有花括号）,实际上，这不是一个问题，因为@onExit，在这种情况下没有任何用处。

```objective-c
// 在一些需要配对使用的地方
{
      [objectLock lock];       // 1
      @onExit {
          [objectLock unlock]; // 3
      };
      // do some stuff         //2
}
```

```objective-c
  // 在一些资源需要处理的情况
rac_propertyAttributes *attributes = rac_copyPropertyAttributes(property);
if (attributes != NULL) {
    @onExit {
        free(attributes);
    };
    // do some stuff using attributes
}
```

下面我们来看看这个宏是怎么运作的

```objective-c
#define onExit \
    rac_keywordify \
    __strong rac_cleanupBlock_t metamacro_concat(rac_exitBlock_, __LINE__) __attribute__((cleanup(rac_executeCleanupBlock), unused)) = ^
  
// 上一篇文章介绍过rac_keywordify的左右，这里直接简化掉
typedef void (^rac_cleanupBlock_t)();

static inline void rac_executeCleanupBlock (__strong rac_cleanupBlock_t *block) {
    (*block)();
}

// 这里约定当前行是第66行 (使用当前行是避免在多个onExit时，变量名冲突)
#define onExit \
__strong rac_cleanupBlock_t rac_exitBlock_66 __attribute__((cleanup(rac_executeCleanupBlock), unused)) = ^

// 最后
{
   @onExit {
         // do some stuff
   };
}
// 展开后
{
  __strong rac_cleanupBlock_t rac_exitBlock_66 __attribute__((cleanup(rac_executeCleanupBlock), unused)) = ^{
     // do some stuff
  }；
}
```

`__attribute__((cleanup(...)))`，用于修饰一个变量，**在它的作用域结束时可以自动执行一个指定的方法**

这样@onExit每次都是在作用域后执行。

这里使用sunnyxx的例子,理解起来可能方便一点

```objective-c
// 指定一个cleanup方法，注意入参是所修饰变量的地址，类型要一样
// 对于指向objc对象的指针(id *)，如果不强制声明__strong默认是__autoreleasing，造成类型不匹配
static void stringCleanUp(__strong NSString **string) {
    NSLog(@"%@", *string);
}
// 在某个方法中：
{
    __strong NSString *string __attribute__((cleanup(stringCleanUp))) = @"sunnyxx";
} // 当运行到这个作用域结束时，自动调用stringCleanUp
```

### metamacro_concat

把两个参数名拼接起来

```
#define metamacro_concat_(A, B) A ## B
```

该宏的用法

```c
//宏名的拼接
metamacro_concat(metamacro_at, N)(__VA_ARGS__)
// N=3
 metamacro_at3(__VA_ARGS__) 
```

### metamacro_at

返回第N个可变参数，至少提供N+1个可变参数(从0开始)

原理：通过调整可变参数的位置（定义20个对应的宏，例如metamacro_at2），把第N-1个参数去掉，再获取第一个参数，即为第N个可变参数

```objective-c
#define metamacro_at(N, ...) metamacro_concat(metamacro_at, N)(__VA_ARGS__)

// 假设N=2
metamacro_at(2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19)

metamacro_at2(-1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19)
  
#define metamacro_at2(_0, _1, ...) metamacro_head(__VA_ARGS__)
metamacro_head(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19)

#define metamacro_head(...) metamacro_head_(__VA_ARGS__, 0)
 // 返回可变参数的第一个参数
metamacro_head(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19，0)

#define metamacro_head_(FIRST, ...) FIRST
1 
```

该宏的几个用法：

返回可变参数的个数

```objective-c
//  假设可变参数的长度为x,第20个参数的值是y
// 那么 y=20-(20-x) 即 y=x 即第20个参数，即使可变参数的长度
#define metamacro_argcount(...) \
        metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
```

返回VAL-1 (只不过是在编译时确定)，在元编程中处理数组index和count问题

```c
#define metamacro_dec(VAL) \
        metamacro_at(VAL, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19)
```

返回VAL+1

```c
#define metamacro_inc(VAL) \
        metamacro_at(VAL, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21)
```

判断N是不是偶数

```
#define metamacro_is_even(N) \
        metamacro_at(N, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1)
```

对0和1取非

```
#define metamacro_not(B) metamacro_at(B, 1, 0)
```

### metamacro_if_eq(A, B)(expression1)(expression2)

如果A = B,那么expression1展开，否则expression2展开

A和B是0到20的数字，另外B>=A

用于在编译时确定执行那个表达式

```c
// 展开后是 true
metamacro_if_eq(0, 0)(true)(false)

// 展开后是 false
metamacro_if_eq(0, 1)(true)(false)
```

我们该宏是怎么展开的

```objective-c
metamacro_if_eq(1, 1)(true)(false)

#define metamacro_if_eq(A, B) metamacro_concat(metamacro_if_eq, A)(B)

metamacro_if_eq1(1)(true)(false)
  
#define metamacro_if_eq1(VALUE) metamacro_if_eq0(metamacro_dec(VALUE))
 metamacro_if_eq0(metamacro_dec(1))(true)(false)
  
// metamacro_dec在上面已做个解释 用于在编译器 获得VAL-1的值
metamacro_if_eq0(0)(true)(false)
  
#define metamacro_if_eq0(VALUE) metamacro_concat(metamacro_if_eq0_, VALUE)
metamacro_if_eq0_0(true)(false)

#define metamacro_if_eq0_0(...) __VA_ARGS__ metamacro_consume_
true metamacro_consume_(false) 
  
#define metamacro_consume_(...) // nothing
true
  
// 同理 如果>1
metamacro_if_eq0_2(true)(false)

#define metamacro_if_eq0_2(...) metamacro_expand_
metamacro_expand_(false)  
  
#define metamacro_expand_(...) __VA_ARGS__
false 
```



### 参考RACObserve(TARGET, KEYPATH)实现一个可自动提示的宏

小知识：逗号表达式

```c
int a = 0, b = 0; 
a = 1, b = 2; // 单行 多个表达式
int c = (a, b); // c将被b赋值，而a是一个未使用的值，编译器会给出warning,强转成void,便可消除
```

```objective-c
// 把 keyPath 变量名 转成 char *字符串，并且自动提示
#define kLTAutoHintMacro(TARGET, KEYPATH) ((void)TARGET.KEYPATH, #KEYPATH)
// 示例，这里obj可自动提示
__unused char *test=kLTAutoHintMacro(self, obj); // test="obj"
```

当我们在敲第二个参数的时候，实际上相当于self.xxxx,所以编辑器会给出正确的代码提示



然后我们看`RACObserve(TARGET, KEYPATH)`一步步是怎么展开的

```objective-c
#define RACObserve(TARGET, KEYPATH) \
	({ \
		_Pragma("clang diagnostic push") \
		_Pragma("clang diagnostic ignored \"-Wreceiver-is-weak\"") \
		__weak id target_ = (TARGET); \
		[target_ rac_valuesForKeyPath:@keypath(TARGET, KEYPATH) observer:self]; \
		_Pragma("clang diagnostic pop") \
	})
 // 去掉消除警告的代码
#define RACObserve(TARGET, KEYPATH) \
	({ \
		__weak id target_ = (TARGET); \
		[target_ rac_valuesForKeyPath:@keypath(TARGET, KEYPATH) observer:self]; \
	})
    
	
```

### 

### @keypath

通过提供一个具体的接收者和KeyPath，@keypath 可以在编译时确定keyPath。这样就可以解决keyPath没有代码提示的问题,避免语法错误，当然这也有助于代码重构。

示例

```objective-c
NSString *UTF8StringPath = @keypath(str.lowercaseString.UTF8String);

// => @"lowercaseString.UTF8String"

NSString *versionPath = @keypath(NSObject, version);

// => @"version"

NSString *lowercaseStringPath = @keypath(NSString.new, lowercaseString);

// => @"lowercaseString"
```

好了，我们看看它是怎么实现该功能的

```objective-c
#define keypath(...) \
    metamacro_if_eq(1, metamacro_argcount(__VA_ARGS__))(keypath1(__VA_ARGS__))(keypath2(__VA_ARGS__))
// 根据上文我们知道metamacro_if_eq，如果可变参数为1，则展开keypath1(__VA_ARGS__)，否则展开为keypath2(__VA_ARGS__)

#define keypath1(PATH) (((void)(NO && ((void)PATH, NO)), strchr(# PATH, '.') + 1))

#define keypath2(OBJ, PATH) (((void)(NO && ((void)OBJ.PATH, NO)), # PATH))

// 我们已介绍过，其中的逗号表达式用于代码提示，简化后
#define keypath1(PATH) strchr(# PATH, '.') + 1

//  char * strchr (const char *str, int c);
// strchr()函数：返回 str 字符串中第一次出现的字符 c 的地址，然后将该地址返回
// 即返回第一个c(包括c)之后的字符串
// 这里keypath1用于去掉变量名 例如，@keypath(str.lowercaseString.UTF8String); 去掉str.

#define keypath2(OBJ, PATH) #PATH
```

###  RAC(TARGET, ...)

我们看该宏是怎么实现位于等号左侧的？

```objective-c
RAC(self,obj)=[RACSignal return:nil];

// 展开后
[[RACSubscriptingAssignmentTrampoline alloc] initWithTarget:(self) nilValue:(((void *)0))][@(((void)(__objc_no && ((void)self.obj, __objc_no)), "obj"))] =[RACSignal return:((void *)0)];

// 简化掉用于自动提示的代码
[[RACSubscriptingAssignmentTrampoline alloc] initWithTarget:(self) nilValue:(((void *)0))][@"obj"] =[RACSignal return:((void *)0)];

// 等价于
RACSubscriptingAssignmentTrampoline *trampoline=[[RACSubscriptingAssignmentTrampoline alloc] initWithTarget:(self) nilValue:(((void *)0))];
trampoline[@"obj"]=[RACSignal return:nil];// 等价于 setObject:forKeyedSubscript:
```

### metamacro_foreach

把可变参数以某种形式展开，例如以逗号作为分割展开

```objective-c
metamacro_foreach(RACTuplePack_object_or_ractuplenil,, @"first",@"second")

// 展开后
@"first", @"second",
```

然后我们一步步分析，展开思路

```objective-c

metamacro_foreach(RACTuplePack_object_or_ractuplenil,, @"first",@"second")

// 和metamacro_foreach_cxt 相同，除了没有提供Context参数，只有参数所在的索引和参数传递给MACRO
// 这里是 下面提到的 RACTuplePack_object_or_ractuplenil(1,@"second")
#define metamacro_foreach(MACRO, SEP, ...) \
        metamacro_foreach_cxt(metamacro_foreach_iter, SEP, MACRO, __VA_ARGS__)

// metamacro_foreach_cxt 负责把参数分隔开，参数和索引对应， 共20个类似的宏
metamacro_foreach_cxt(metamacro_foreach_iter, , RACTuplePack_object_or_ractuplenil, __VA_ARGS__)
  
// 对于每个连续的可变参数(最多20个)，传递第index个参数给MACRO,CONTEXT,和参数本身给MACRO,这样最后的结果是MACRO的相邻的参数以SEP分割
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(__VA_ARGS__))(MACRO, SEP, CONTEXT, __VA_ARGS__) 
  
metamacro_foreach_cxt2(metamacro_foreach_iter, , RACTuplePack_object_or_ractuplenil, __VA_ARGS__) 
  
#define metamacro_foreach_cxt2(MACRO, SEP, CONTEXT, _0, _1) \
    metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) \
    SEP \
    MACRO(1, CONTEXT, _1)

 metamacro_foreach_cxt1(metamacro_foreach_iter, , RACTuplePack_object_or_ractuplenil,  @"first") \
   \
  MACRO(1, CONTEXT, @"second")
  
#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)
  
metamacro_foreach_iter(0,RACTuplePack_object_or_ractuplenil,@"first") \
   \
metamacro_foreach_iter(1, RACTuplePack_object_or_ractuplenil, @"second")
 
// 转换成自定义分割样式
#define metamacro_foreach_iter(INDEX, MACRO, ARG) MACRO(INDEX, ARG)
  
RACTuplePack_object_or_ractuplenil(0,@"first")
\
RACTuplePack_object_or_ractuplenil(1,@"second")
  
#define RACTuplePack_object_or_ractuplenil(INDEX, ARG) \
    (ARG) ?: RACTupleNil.tupleNil,
  
@"first", @"second", 
```

基本思路是：计算可变参数的个数，通过递归，拆分参数，并根据自定义样式分割。