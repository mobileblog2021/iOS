```
dispatch_async
_dispatch_continuation_async
_dispatch_continuation_async2
_dispatch_async_f2
_dispatch_queue_push
_dispatch_queue_push_slow
_dispatch_trace_queue_push_inline
_dispatch_queue_push_inline
(&(dq)->do_vtable->_os_obj_vtable)->do_wakeup(dp, pp, flags)
_dispatch_queue_wakeup
_dispatch_queue_class_wakeup_enqueue
_dispatch_queue_push_slow



_dispatch_queue_push_slow
_dispatch_root_queues_init_once
_dispatch_root_queues_init_workq
_dispatch_worker_thread3
_dispatch_root_queue_drain
_dispatch_client_callout
_dispatch_async_redirect_invoke
_dispatch_client_callout
_dispatch_call_block_and_release
_block_invoke
```







```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_async_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func,
		pthread_priority_t pp, dispatch_block_flags_t flags)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc_cacheonly();
	uintptr_t dc_flags = DISPATCH_OBJ_CONSUME_BIT;

	if (!fastpath(dc)) {
		return _dispatch_async_f_slow(dq, ctxt, func, pp, flags, dc_flags);
	}

	_dispatch_continuation_init_f(dc, dq, ctxt, func, pp, flags, dc_flags);
	_dispatch_continuation_async2(dq, dc, false);
}
```

```c
// 把func 和其他标志位信息封装为dispatch_continuation_t对象
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_init_f(dispatch_continuation_t dc,
		dispatch_queue_class_t dqu, void *ctxt, dispatch_function_t func,
		pthread_priority_t pp, dispatch_block_flags_t flags, uintptr_t dc_flags)
{
	dc->dc_flags = dc_flags;
	dc->dc_func = func;
	dc->dc_ctxt = ctxt;
	_dispatch_continuation_voucher_set(dc, dqu, flags);
	_dispatch_continuation_priority_set(dc, pp, flags);
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_async2(dispatch_queue_t dq, dispatch_continuation_t dc,
		bool barrier)
{
     // _width > 1 && _width < DISPATCH_QUEUE_WIDTH_POOL;
     // 如果是障碍类型的任务 或 串行队列
	if (fastpath(barrier || !DISPATCH_QUEUE_USES_REDIRECTION(dq->dq_width))) {
		return _dispatch_continuation_push(dq, dc);
	}
	return _dispatch_async_f2(dq, dc);
}
```

```c
DISPATCH_NOINLINE
static void
_dispatch_continuation_push(dispatch_queue_t dq, dispatch_continuation_t dc)
{
	_dispatch_queue_push(dq, dc,
			_dispatch_continuation_get_override_priority(dq, dc));
}

// 将任务入队
DISPATCH_NOINLINE
void
_dispatch_queue_push(dispatch_queue_t dq, dispatch_object_t dou,
		pthread_priority_t pp)
{
	_dispatch_assert_is_valid_qos_override(pp);
    // dx_type(dq) 等价于 (&(dq)->do_vtable->_os_obj_vtable) 
    // 如果 是全局队列
	if (dx_type(dq) == DISPATCH_QUEUE_GLOBAL_ROOT_TYPE) {
#if DISPATCH_USE_KEVENT_WORKQUEUE
		dispatch_deferred_items_t ddi = _dispatch_deferred_items_get();
		if (unlikely(ddi && !(ddi->ddi_stashed_pp &
				(dispatch_priority_t)_PTHREAD_PRIORITY_FLAGS_MASK))) {
			dispatch_assert(_dispatch_root_queues_pred == DLOCK_ONCE_DONE);
			return _dispatch_trystash_to_deferred_items(dq, dou, pp, ddi);
		}
#endif
#if HAVE_PTHREAD_WORKQUEUE_QOS
		// can't use dispatch_once_f() as it would create a frame
		if (unlikely(_dispatch_root_queues_pred != DLOCK_ONCE_DONE)) {
			return _dispatch_queue_push_slow(dq, dou, pp);
		}
		if (_dispatch_need_global_root_queue_override(dq, pp)) {
			return _dispatch_root_queue_push_override(dq, dou, pp);
		}
#endif
	}
	_dispatch_queue_push_inline(dq, dou, pp, 0);
}
```

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_trace_queue_push_inline(dispatch_queue_t dq, dispatch_object_t _tail,
		pthread_priority_t pp, dispatch_wakeup_flags_t flags)
{
	if (slowpath(DISPATCH_QUEUE_PUSH_ENABLED())) {
		struct dispatch_object_s *dou = _tail._do;
		_dispatch_trace_continuation(dq, dou, DISPATCH_QUEUE_PUSH);
	}
	_dispatch_introspection_queue_push(dq, _tail);
	_dispatch_queue_push_inline(dq, _tail, pp, flags);
}
```

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_queue_push_inline(dispatch_queue_t dq, dispatch_object_t _tail,
		pthread_priority_t pp, dispatch_wakeup_flags_t flags)
{
	struct dispatch_object_s *tail = _tail._do;
	bool override = _dispatch_queue_need_override(dq, pp);
	if (flags & DISPATCH_WAKEUP_SLOW_WAITER) {
		// when SLOW_WAITER is set, we borrow the reference of the caller
		if (unlikely(_dispatch_queue_push_update_tail(dq, tail))) {
			_dispatch_queue_push_update_head(dq, tail, true);
			flags = DISPATCH_WAKEUP_SLOW_WAITER | DISPATCH_WAKEUP_FLUSH;
		} else if (override) {
			flags = DISPATCH_WAKEUP_SLOW_WAITER | DISPATCH_WAKEUP_OVERRIDING;
		} else {
			flags = DISPATCH_WAKEUP_SLOW_WAITER;
		}
	} else {
		if (override) _dispatch_retain(dq);
		if (unlikely(_dispatch_queue_push_update_tail(dq, tail))) {
			_dispatch_queue_push_update_head(dq, tail, override);
			flags = DISPATCH_WAKEUP_CONSUME | DISPATCH_WAKEUP_FLUSH;
		} else if (override) {
			flags = DISPATCH_WAKEUP_CONSUME | DISPATCH_WAKEUP_OVERRIDING;
		} else {
			return;
		}
	}
   //  (&(dq)->do_vtable->_os_obj_vtable)->do_wakeup(dp, pp, flags)
	return dx_wakeup(dq, pp, flags);
}
```

```c
struct dispatch_queue_s {
    
	const struct dispatch_queue_vtable_s *do_vtable 
      
    // ...
};

__attribute__((section("__DATA,__objc_data"), used))
const struct dispatch_queue_extra_vtable_s
_OS_dispatch_queue_concurrent_vtable= { 
    .do_type = DISPATCH_QUEUE_CONCURRENT_TYPE,
	.do_kind = "concurrent-queue",
	.do_dispose = _dispatch_queue_dispose,
	.do_suspend = _dispatch_queue_suspend,
	.do_resume = _dispatch_queue_resume,
    .do_finalize_activation = _dispatch_queue_finalize_activation,
	.do_invoke = _dispatch_queue_invoke,
	.do_wakeup = _dispatch_queue_wakeup, // 这里
	.do_debug = dispatch_queue_debug,
	.do_set_targetq = _dispatch_queue_set_target_queue, 
 };
```

```c
DISPATCH_NOINLINE
void
_dispatch_queue_wakeup(dispatch_queue_t dq, pthread_priority_t pp,
		dispatch_wakeup_flags_t flags)
{
	dispatch_queue_wakeup_target_t target = DISPATCH_QUEUE_WAKEUP_NONE;

	if (_dispatch_queue_class_probe(dq)) {
		target = DISPATCH_QUEUE_WAKEUP_TARGET;
	}
	if (target) {
		return _dispatch_queue_class_wakeup(dq, pp, flags, target);
	} else if (pp) {
		return _dispatch_queue_class_override_drainer(dq, pp, flags);
	} else if (flags & DISPATCH_WAKEUP_CONSUME) {
		return _dispatch_release_tailcall(dq);
	}
}
```

```c

    static NSString *specificKey=@"com.shenhualxt.custom_queue";
    NSString *queue_label=specificKey;
    
    dispatch_queue_t queue=dispatch_queue_create(queue_label.UTF8String, DISPATCH_QUEUE_CONCURRENT);
    
    CFStringRef specificValue=(__bridge CFStringRef)queue_label;
    
    dispatch_queue_set_specific(queue, specificKey.UTF8String, (void *)specificValue, (dispatch_function_t)CFRelease);
    
    BOOL isCurrentQueue=((__bridge NSString *)dispatch_get_specific("com.shenhualxt.custom_queue")).length>0;
    if (isCurrentQueue) {
        NSLog(@"是同一个队列");
    }
    dispatch_async(queue, ^{
        BOOL isCurrentQueue=((__bridge NSString *)dispatch_get_specific("com.shenhualxt.custom_queue")).length>0;
        if (isCurrentQueue) {
            NSLog(@"是同一个队列");
        }
       NSString *queue_label_name= ((__bridge NSString *)dispatch_get_specific(specificKey.UTF8String));
       NSLog(@"%@",queue_label_name);
    });

```