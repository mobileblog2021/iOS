### RACTupleUnpack

该宏用于展开RACTuple的子元素

```objective-c
// 用法
RACTupleUnpack(NSString *string, NSNumber *num) = [RACTuple tupleWithObjects:@"foo", @5, nil];
```

我们一步步来分析，该宏是怎么生效的

```objective-c
// 展开示例
RACTupleUnpack(NSString *first,NSString *second)

#define RACTupleUnpack_(...) \ // metamacro_foreach 前几篇文章中有介绍，这里直接展开
    metamacro_foreach(RACTupleUnpack_decl,, __VA_ARGS__) \ 
    \
    int RACTupleUnpack_state = 0; \
    \
    RACTupleUnpack_after: \
        ; \
        metamacro_foreach(RACTupleUnpack_assign,, __VA_ARGS__) \
        if (RACTupleUnpack_state != 0) RACTupleUnpack_state = 2; \
        \
        while (RACTupleUnpack_state != 2) \
            if (RACTupleUnpack_state == 1) { \
                goto RACTupleUnpack_after; \
            } else \
                for (; RACTupleUnpack_state != 1; RACTupleUnpack_state = 1) \
                    [RACTupleUnpackingTrampoline trampoline][ @[ metamacro_foreach(RACTupleUnpack_value,, __VA_ARGS__) ] ]
    
// __strong id RACTupleUnpack36_var1;
#define RACTupleUnpack_decl(INDEX, ARG)  __strong id RACTupleUnpack_decl_name(INDEX);                
//  RACTupleUnpack36_var1      
#define RACTupleUnpack_decl_name(INDEX) \
    metamacro_concat(metamacro_concat(RACTupleUnpack, __LINE__), metamacro_concat(_var, INDEX))
// RACTupleUnpack_after36
#define RACTupleUnpack_after metamacro_concat(RACTupleUnpack_after, __LINE__)
       
// __strong NSString *second = RACTupleUnpack36_var1;
#define RACTupleUnpack_assign(INDEX, ARG) __strong ARG = RACTupleUnpack_decl_name(INDEX);
// RACTupleUnpack_state36
#define RACTupleUnpack_state metamacro_concat(RACTupleUnpack_state, __LINE__)         
          
// 展开后        
__strong id RACTupleUnpack36_var0;
__strong id RACTupleUnpack36_var1;

int RACTupleUnpack_state36 = 0;
RACTupleUnpack_after36:; // 定义了一个标号

__strong NSString *first = RACTupleUnpack36_var0;
__strong NSString *second = RACTupleUnpack36_var1;

if (RACTupleUnpack_state36 != 0)
    RACTupleUnpack_state36 = 2;
while (RACTupleUnpack_state36 != 2){
    if (RACTupleUnpack_state36 == 1) {
        goto RACTupleUnpack_after36; // 这里的goto用法有待研究
    } else
        for (; RACTupleUnpack_state36 != 1; RACTupleUnpack_state36 = 1){
            // 解包操作  (这里的局部变量，只是方便理解)
            NSValue *var0=[NSValue valueWithPointer:&RACTupleUnpack36_var0];
            NSValue *var1=[NSValue valueWithPointer:&RACTupleUnpack36_var1];
            NSArray *array=@[ @"first", (@"second"), ];
            RACTuple *tuple=[RACTuple tupleWithObjectsFromArray:array];
            // RACTupleUnpackingTrampoline设计出来用来实现神奇的RACTupleUnpack( ) 这个宏
            // setObject:forKeyedSubscript: 
            [RACTupleUnpackingTrampoline trampoline][ @[ var0, var1,] ]= (tuple);
        }
}
          
                
```