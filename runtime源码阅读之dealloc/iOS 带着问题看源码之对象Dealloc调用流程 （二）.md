# iOS 带着问题看源码之对象Dealloc调用流程 （二）

❓❓❓： Person对象的dealloc方法为什么无需调用[super dealloc]
#### 演示代码
```objective-c
@interface Person : NSObject
@end

@implementation Person

- (void)dealloc {
}

@end
    
int main(int argc, const char * argv[]) {
    Person *objc = [Person new];
    return 0;
}
```
通过clang将代码编译，分析llvm的中间语言，通过以下命令将代码编译成中间语言：

    clang -S -fobjc-arc -emit-llvm main.m -o main.ll
    
  mail.ll 文件部分内容如下
```objective-c
define internal void @"\01-[Person dealloc]"(%0* %0, i8* %1) #0 {
  %3 = alloca %0*, align 8
  %4 = alloca i8*, align 8
  %5 = alloca %struct._objc_super, align 8
  store %0* %0, %0** %3, align 8
  store i8* %1, i8** %4, align 8
  %6 = load %0*, %0** %3, align 8
  %7 = bitcast %0* %6 to i8*
  %8 = getelementptr inbounds %struct._objc_super, %struct._objc_super* %5, i32 0, i32 0
  store i8* %7, i8** %8, align 8
  %9 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_SUP_REFS_$_", align 8
  %10 = bitcast %struct._class_t* %9 to i8*
  %11 = getelementptr inbounds %struct._objc_super, %struct._objc_super* %5, i32 0, i32 1
  store i8* %10, i8** %11, align 8
  %12 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_, align 8, !invariant.load !9
  // 编译器自动插入的[super dealloc]
  call void bitcast (i8* (%struct._objc_super*, i8*, ...)* @objc_msgSendSuper2 to void (%struct._objc_super*, i8*)*)(%struct._objc_super* %5, i8* %12)
  ret void
}

define i32 @main(i32 %0, i8** %1) #1 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i8**, align 8
  %6 = alloca %0*, align 8
  store i32 0, i32* %3, align 4
  store i32 %0, i32* %4, align 4
  store i8** %1, i8*** %5, align 8
  %7 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %8 = bitcast %struct._class_t* %7 to i8*
  %9 = call i8* @objc_opt_new(i8* %8)
  %10 = bitcast i8* %9 to %0*
  store %0* %10, %0** %6, align 8
  store i32 0, i32* %3, align 4
  %11 = bitcast %0** %6 to i8**
  // 自动插入storeStrong(ojbect, nil) 实现release操作
  call void @llvm.objc.storeStrong(i8** %11, i8* null) #2
  %12 = load i32, i32* %3, align 4
  ret i32 %12
}
```
即编译器在变成中中间语言时，自动插入了[super dealloc]和 对象释放的release代码
所以person对象的释放过程等价于
```objective-c
int main(int argc, const char * argv[]) {
    Person *person = [Person new];
    objc_storeStrong(person, nil);
    return 0;
}
```

我们通过编译器自动插入的storeStrong向下分析

```objective-c
// 函数会对新的对象进行 retain 操作，对老的对象进行 release 操作
// 用于对__strong 变量的赋值 和 对象的release操作
void
objc_storeStrong(id *location, id obj)
{
    id prev = *location;
    if (obj == prev) {
        return;
    }
    objc_retain(obj);
    *location = obj;
    objc_release(prev);
}
```
等价于
```objective-c
int main(int argc, const char * argv[]) {
    Person *person = [Person new];
    objc_release(person)
    return 0;
}
```
objc_release(id) --> objc_object::release() --> objc_object::rootRelease()
```objective-c
ALWAYS_INLINE bool
objc_object::rootRelease(bool performDealloc, objc_object::RRVariant variant)
{
   
   // 省略其他check
   // don't check newisa.fast_rr; we already called any RR overrides
    uintptr_t carry;
        // ⛵⛵⛵  引用计数 -1
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // don't ClearExclusive()
            goto underflow;
        }
        // ⛵⛵⛵ 如果引用计数 == 0 则释放 isDeallocating = extra_rc == 0 && has_sidetable_rc == 0；
        if (slowpath(newisa.isDeallocating()))
        goto deallocate;

deallocate:
    // Really deallocate.

    ASSERT(newisa.isDeallocating());
    ASSERT(isa.isDeallocating());

    if (slowpath(sideTableLocked)) sidetable_unlock();

    __c11_atomic_thread_fence(__ATOMIC_ACQUIRE);

    if (performDealloc) {
         // ⛵⛵⛵ 调用 当前对象的 dealloc方法
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(dealloc));
    }
    return true;
}
```
即引用计数变为零后，调用对象的dealloc方法，即[person dealloc], 在上面我们知道
```objective-c
@implementation Person

- (void)dealloc {
   objc_msgSuper(self, @selector(dealloc));
}
@end
```
```objective-c
// NSObject.mm
@implementation NSObject

- (void)dealloc {
    _objc_rootDealloc(self);
}

@end
```
