### DISPATCH_DECL

GCD 中对变量的定义大多遵循如下格式:

```c
#define DISPATCH_DECL(name) typedef struct name##_s *name##_t

// 示例：定义了一个 dispatch_queue_t 类型的指针，指向一个 dispatch_queue_s 类型的结构体
typedef struct dispatch_queue_s *dispatch_queue_t;
```

### TSD （线程私有数据 Thread Specific Data）

在多线程程序中，所有线程共享程序中的变量。现在有一全局变量，所有线程都可以使用它，改变它的值。而如果每个线程希望能单独拥有它，那么就需要使用线程存储了。表面上看起来这是一个全局变量，所有线程都可以使用它，而它的值在每一个线程中又是单独存储的。这就是线程存储的意义。

TSD 的作用就是能够在同一个线程的不同函数中被访问。在不同线程中，虽然名字相同，但是获取到的数据随线程不同而不同。

利用POSIX 库提供的 API 来实现 TSD:

```c
// 创建一个类型为 pthread_key_t 类型的变量。  
pthread_key_t   key;
// 调用 pthread_key_create() 来创建该变量。该函数有两个参数，第一个参数就是上面声明的 pthread_key_t 变量，第二个参数是一个清理函数，用来在线程释放该线程存储的时候被调用。该函数指针可以设成 NULL ，这样系统将调用默认的清理函数。
int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));

// 第一个为前面声明的 pthread_key_t 变量，第二个为 void* 变量，这样你可以存储任何类型的值。
int pthread_setspecific(pthread_key_t key, const void *value);

// 第一个为前面声明的 pthread_key_t 变量，该函数返回 void * 类型的值
void *pthread_getspecific(pthread_key_t key);
```

下面是一个如何使用线程存储的例子：

```c
#include <stdio.h>
#include <pthread.h>
pthread_key_t   key;
void echomsg(int t)
{
        printf("destructor excuted in thread %d,param=%d\n",pthread_self(),t);
}
void * child1(void *arg)
{
        int tid=pthread_self();
        printf("thread %d enter\n",tid);
        pthread_setspecific(key,(void *)tid);
        sleep(2);
        printf("thread %d returns %d\n",tid,pthread_getspecific(key));
        sleep(5);
}
void * child2(void *arg)
{
        int tid=pthread_self();
        printf("thread %d enter\n",tid);
        pthread_setspecific(key,(void *)tid);
        sleep(1);
        printf("thread %d returns %d\n",tid,pthread_getspecific(key));
        sleep(5);
}
int main(void)
{
        int tid1,tid2;
        printf("hello\n");
        pthread_key_create(&key,echomsg);
        pthread_create(&tid1,NULL,child1,NULL);
        pthread_create(&tid2,NULL,child2,NULL);
        sleep(10);
        pthread_key_delete(key);
        printf("main thread exit\n");
        return 0;
}
```

### fastpath && slowpath

```c
#define fastpath(x) ((typeof(x))__builtin_expect((long)(x), ~0l))
#define slowpath(x) ((typeof(x))__builtin_expect((long)(x), 0l))
```

`__builtin_expect` 可以指定某个值的预期值。这样可以让编译器对相关的分支代码进行优化。

在现代的处理器设计中，因为 cpu 和内存的速度不匹配。所以现在的 cpu 会读取多条指令并执行（多流水线）。比如，遇到分支时，它会同时计算分支的条件，并选择第一个分支执行。

```c
// __builtin_expect(x, 5) 代码告诉编译器，x 的值具有很大的概率是5。
switch (__builtin_expect(x, 5)) {
  default: break;
  case 0:  // ...
  case 3:  // ...
  case 5:  // ...
}

// 如果没有任何优化，处理器会经常进入default分支执行，导致资源浪费。在这种情况下，编译后的代码会调整分支的顺序以达到提高资源效率的最优化。
switch (__builtin_expect(x, 5)) {
  case 5:  // ...
  default: break;
  case 0:  // ...
  case 3:  // ...
}
```

在上面定义的两个宏中，`fastpath(x)` 依然返回 x，只是告诉编译器 x 的值一般不为 0，从而编译器可以进行优化。同理，`slowpath(x)` 表示 x 的值很可能为 0，希望编译器进行优化。

### Quality of Service(QoS)

这是在iOS8之后提供的新功能，苹果提供了几个Quality of Service枚举来使用:user interactive, user initiated, utility 和 background，通过这告诉系统我们在进行什么样的工作，然后系统会通过合理的资源控制来最高效的执行任务代码，其中主要涉及到CPU调度的优先级、IO优先级、任务运行在哪个线程以及运行的顺序等等，我们通过一个抽象的Quality of Service参数来表明任务的意图以及类别。

```objective-c
// 与用户交互的任务，这些任务通常跟UI级别的刷新相关，比如动画，这些任务需要在一瞬间完成
NSQualityOfServiceUserInteractive
  
// 由用户发起的并且需要立即得到结果的任务，比如滑动scroll view时去加载数据用于后续cell的显示，这些任务通常跟后续的用户交互相关，在几秒或者更短的时间内完成
NSQualityOfServiceUserInitiated 
  
// 一些可能需要花点时间的任务，这些任务不需要马上返回结果，比如下载的任务，这些任务可能花费几秒或者几分钟的时间
NSQualityOfServiceUtility
  
// 这些任务对用户不可见，比如后台进行备份的操作，这些任务可能需要较长的时间，几分钟甚至几个小时
NSQualityOfServiceBackground
  
// 优先级介于user-initiated 和 utility，当没有 QoS信息时默认使用，开发者不应该使用这个值来设置自己的任务
NSQualityOfServiceDefault
  
```

| Global queue                       | Corresponding Qos class |
| ---------------------------------- | ----------------------- |
| Main thread                        | User-interactive        |
| DISPATCH_QUEUE_PRIORITY_HIGH       | User-initiated          |
| DISPATCH_QUEUE_PRIORITY_DEFAULT    | Default                 |
| DISPATCH_QUEUE_PRIORITY_LOW        | Utility                 |
| DISPATCH_QUEUE_PRIORITY_BACKGROUND | Background              |



### AutoreleaseFrequency

作为Swift3出现的新属性,这个属性应该总是为.workItem

**历史原因:AutoreleaseFrequency是为了弥补以前GCD当线程不活跃时,会在开发者无法掌控的时间自动往自动释放池释放对象,而这种无法掌握的事情缺点很多,实际开发中，开发者要么自己手动创建释放池释放调度对象，要么出现内存悬挂。于是Apple使用新的workitem来解决手动释放的问题,其实2个选项为了适配旧的代码,never是GCD不会为你管理自动释放池,可能为一些特殊项目使用,inherit是旧的模式，适配旧代码**

```swift
// 从目标队列继承 ，这是手动创建队列的默认值
.inherit

// 为每个提交的任务 ，创建和销毁一个自动释放池
.workItem

// GCD 不管理自动释放池
.never: GCD doesn't manage autorelease pools for you
```

### 一些简写

`dq`: dispatch_queue

`qos`: quality of service