### # 将宏中的参数符串化

这个字符串化会将参数中的所有字符都实现字符串化，包括引号。如果参数中间有很多空格，字符串化之后将会只用一个空格代替。

```c
#define WARN_IF(EXP)  fprintf (stderr, "Warning: " #EXP "\n")

WARN_IF (x == 0); 
// 展开后
fprintf (stderr, "Warning: " "x == 0" "\n"); 
```

### #@表示返回是一个字符const char

```
#define ToChar(x) #@x
char a = ToChar(1);结果就是a='1';
```

#####操作符可以实现宏中token的连接

```c
<token> ## <token>

#define type i##nt
type a; // 等价于 int a

#define DECLARE_AND_SET(type, varname, value) type varname = value; type orig_##varname = varname
DECLARE_AND_SET( int, area, 2 * 6 );
// 等价于
int area = 2 * 6; int orig_area = area;
```

### `__VA_ARGS__` 多参数宏（Variadic Macros)

该标识符用来表示可变参数 ， … 用在宏的名称中

```c
#define eprintf(...) fprintf (stderr, __VA_ARGS__)  
```

特别：`##__VA_ARGS__ `,如果后方文本(即可变参数)为空，那么它会将前面一个逗号吃掉

```objective-c
#define LTLog(format,...) NSLog((format), ##__VA_ARGS__);

// 这样也不会报错，因为可变参数为空，前面那个，也被吃掉了
LTLog(@"test");
```

###@  OC中将char *转成NSString（其实这个只是拼接）

```objective-c
#define test(c) @c 
NSString *b=test("aaa");
```

