在苹果文档中介绍的global queues（意味着在对于整个App的全局性，并不需要初始化），在libdispatch 中称作root queue。

注意队列不是线程，单个队列可能在多个工作线程中执行，一个工作线程中也可能有多个队列。

dispatch_queue_s 使用了两个宏来定义，通过这种方式模拟C++中的类和子类

```c
struct dispatch_queue_s {
	_DISPATCH_QUEUE_HEADER(queue);
	DISPATCH_QUEUE_CACHELINE_PADDING; // for static queues only
} DISPATCH_QUEUE_ALIGN;
```

```c

struct dispatch_queue_s {
	// _DISPATCH_QUEUE_HEADER(queue); - from queue_internal.h
	struct os_mpsc_queue_s _as_oq[0]; 
	// DISPATCH_OBJECT_HEADER(queue); 
    struct dispatch_object_s _as_do[0]; 
    //_DISPATCH_OBJECT_HEADER(queue)
    struct _os_object_s _as_os_obj[0]; 

	// OS_OBJECT_STRUCT_HEADER(dispatch_##x); 
    // _OS_OBJECT_HEADER(const void *_objc_isa, do_ref_cnt, do_xref_cnt); 
    const void *_objc_isa; /* must be pointer-sized */ 
    int volatile do_ref_cnt;  // reference count 内部引用计数
    int volatile do_xref_cnt;  // cross reference count 外部引用计数
    // object operations table
    // 定义了对象在不同操作下该执行的方法，
    // 比如在的 probe 操作下，实际上会执行 _dispatch_queue_wakeup_global 方法
	const struct dispatch_queue_vtable_s *do_vtable 
    
    // pointer to next object (i.e. linked list)
	struct dispatch_queue_s *volatile do_next;
     // Actual target of object (one of the root queues)
	struct dispatch_queue_s *do_targetq; 
     // context 
	void *do_ctxt; 
    // Set with dispatch_set_finalizer[_f]
	void *do_finalizer
      
	// _OS_MPSC_QUEUE_FIELDS(dq, dq_state); 
     #define _OS_MPSC_QUEUE_FIELDS(ns, __state_field__) 
     // pointer to first item on dispatch queue (for remove)
	struct dispatch_object_s *volatile dq_items_head;
     // pointer to last item on dispatch queue (for insert)
	struct dispatch_object_s *volatile dq_items_tail;
     // Serial # (1-12)
	unsigned long dq_serialnum; 
	union { 
		uint64_t volatile dq_state; 
		DISPATCH_STRUCT_LITTLE_ENDIAN_2( 
			dispatch_lock dq_state_lock, 
			uint32_t dq_state_bits 
		); 
	}; /* needs to be 64-bit aligned */ 
	/* LP64 global queue cacheline boundary */ 
     //  User-defined; obtain with get_label,for debugging
	const char *dq_label; 
	voucher_t dq_override_voucher; 
    // 队列优先级
	dispatch_priority_t dq_priority; 
	dispatch_priority_t volatile dq_override; 
     
    // Used for dispatch_queue_set/get_specific
	dispatch_queue_t dq_specific_q; 
	union {	
		uint32_t volatile dq_atomic_flags; 
         // 小端
		DISPATCH_STRUCT_LITTLE_ENDIAN_2( 
			uint16_t dq_atomic_bits, 
             // Concurrency "width" (how many objects run in parallel)
             //  of threads in pool
			uint16_t dq_width 
		); 
	}; 
  
    // increment/decrement with dispatch_queue_suspend/resume
	uint32_t dq_side_suspend_cnt; 
	//DISPATCH_INTROSPECTION_QUEUE_HEADER; 
    // introspection builds (-DDISPATCH_INTROSPECTION) only
    TAILQ_ENTRY(dispatch_queue_s) diq_list;   
    dispatch_unfair_lock_s diq_order_top_head_lock; 
    dispatch_unfair_lock_s diq_order_bottom_head_lock; 
    TAILQ_HEAD(, dispatch_queue_order_entry_s) diq_order_top_head; 
    TAILQ_HEAD(, dispatch_queue_order_entry_s) diq_order_bottom_head
	dispatch_unfair_lock_s dq_sidelock
	/* LP64: 32bit hole on LP64 */
      
      
	// DISPATCH_QUEUE_CACHELINE_PADDING; // for static queues only
     //  pads to 64-byte boundary
     // ensure that the structure can fit optimally within the CPU's cache lines
     char _dq_pad[DISPATCH_QUEUE_CACHELINE_PAD]
};
```
```c
dispatch_queue_t
dispatch_queue_create(const char *label, dispatch_queue_attr_t attr)
{
	return _dispatch_queue_create_with_target(label, attr,
			DISPATCH_TARGET_QUEUE_DEFAULT, true);
}


DISPATCH_NOINLINE
static dispatch_queue_t
_dispatch_queue_create_with_target(const char *label, dispatch_queue_attr_t dqa,
		dispatch_queue_t tq, bool legacy)
{
#if DISPATCH_USE_NOQOS_WORKQUEUE_FALLBACK
	// Be sure the root queue priorities are set
	dispatch_once_f(&_dispatch_root_queues_pred, NULL,
			_dispatch_root_queues_init_once);
#endif
	if (!slowpath(dqa)) {
         // 默认创建 串行的，优先级为default属性 的优先级
		dqa = _dispatch_get_default_queue_attr();
	} else if (dqa->do_vtable != DISPATCH_VTABLE(queue_attr)) {
		DISPATCH_CLIENT_CRASH(dqa->do_vtable, "Invalid queue attribute");
	}

	// 标准化参数 (服务质量，过载，目标队列)
	// Step 1: Normalize arguments (qos, overcommit, tq)
	// 
    
    // 服务质量
	qos_class_t qos = dqa->dqa_qos_class;
#if DISPATCH_USE_NOQOS_WORKQUEUE_FALLBACK  // 1
    // 1、是User-interactive(用户交互) 2、用户交互对应队列没有设置优先级
	if (qos == _DISPATCH_QOS_CLASS_USER_INTERACTIVE &&
			!_dispatch_root_queues[
			DISPATCH_ROOT_QUEUE_IDX_USER_INTERACTIVE_QOS].dq_priority) {
         // 调整服务质量为User-initiated(用户发起)
		qos = _DISPATCH_QOS_CLASS_USER_INITIATED;
	}
#endif
	bool maintenance_fallback = false;
#if DISPATCH_USE_NOQOS_WORKQUEUE_FALLBACK
	maintenance_fallback = true;
#endif // DISPATCH_USE_NOQOS_WORKQUEUE_FALLBACK
	if (maintenance_fallback) {
         // 1、服务质量是MAINTENANCE 2、MAINTENANCE对应的队列，没有设置优先级
		if (qos == _DISPATCH_QOS_CLASS_MAINTENANCE &&
				!_dispatch_root_queues[
				DISPATCH_ROOT_QUEUE_IDX_MAINTENANCE_QOS].dq_priority) {
             // 调整服务质量为background
			qos = _DISPATCH_QOS_CLASS_BACKGROUND;
		}
	}

	_dispatch_queue_attr_overcommit_t overcommit = dqa->dqa_overcommit;
    // 指定过过载属性 2，有目标队列
	if (overcommit != _dispatch_queue_attr_overcommit_unspecified && tq) { 
		if (tq->do_targetq) {
             // 不能既是过载，目标队列又不是全局队列
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify both overcommit and "
					"a non-global target queue");
		}
	}
    
    // 1、有目标队列 2、目标队列是全局队列 3、内部引用计数是默认值
	if (tq && !tq->do_targetq &&
			tq->do_ref_cnt == DISPATCH_OBJECT_GLOBAL_REFCNT) {
		// Handle discrepancies between attr and target queue, attributes win
         // 如果attr和目标队列的attr有差异，以attr为准
         // 没有指定过载属性
		if (overcommit == _dispatch_queue_attr_overcommit_unspecified) {
             // 如果优先级中包含overcommit (通过与运算检测)
			if (tq->dq_priority & _PTHREAD_PRIORITY_OVERCOMMIT_FLAG) {
                 // 开启过载
				overcommit = _dispatch_queue_attr_overcommit_enabled;
			} else {
                 // 禁用过载
				overcommit = _dispatch_queue_attr_overcommit_disabled;
			}
		}
         // 属性没有指定服务质量
		if (qos == _DISPATCH_QOS_CLASS_UNSPECIFIED) {
             // 调整目标队列的过载属性，因为在_dispatch_root_queues 数组中，成对的过载和非过载队列相邻，通过指针偏移（+1 或-1），来调整过载属性
			tq = _dispatch_get_root_queue_with_overcommit(tq,
					overcommit == _dispatch_queue_attr_overcommit_enabled);
		} else {
			tq = NULL;
		}
	} else if (tq && !tq->do_targetq) { // 1、有目标队列 2、目标队列是全局队列
		// target is a pthread or runloop root queue, setting QoS or overcommit
		// is disallowed
		if (overcommit != _dispatch_queue_attr_overcommit_unspecified) {
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify an overcommit attribute "
					"and use this kind of target queue");
		}
		if (qos != _DISPATCH_QOS_CLASS_UNSPECIFIED) {
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify a QoS attribute "
					"and use this kind of target queue");
		}
	} else {
         // 没有指定 overcommit
		if (overcommit == _dispatch_queue_attr_overcommit_unspecified) {
			 // Serial queues default to overcommit!    
              // 串行队列过载，并行队列不过载，这就是说创建多少个串行队列，就会创建多少个线程
			overcommit = dqa->dqa_concurrent ?
					_dispatch_queue_attr_overcommit_disabled :
					_dispatch_queue_attr_overcommit_enabled;
		}
	}
    // 目标队列不存在 1、tq参数没有传递 2、有目标队列 ，目标队列是全局队列，内部引用计数是默认值，qos是未指定
	if (!tq) {
         // 如果qos 是未指定，则设置为默认的qos
		qos_class_t tq_qos = qos == _DISPATCH_QOS_CLASS_UNSPECIFIED ?
				_DISPATCH_QOS_CLASS_DEFAULT : qos;
        // 根据qos 获得root queue,并设置为目标队列
		tq = _dispatch_get_root_queue(tq_qos, overcommit ==
				_dispatch_queue_attr_overcommit_enabled);
		if (slowpath(!tq)) {
			DISPATCH_CLIENT_CRASH(qos, "Invalid queue attribute");
		}
	}

	//
	// Step 2: Initialize the queue
	//

	if (legacy) {
		// if any of these attributes is specified, use non legacy classes
         // 如果指定了以下属性，那么不再使用过期的api
		if (dqa->dqa_inactive || dqa->dqa_autorelease_frequency) {
			legacy = false;
		}
	}

	const void *vtable;
     // 存储队列的一些标志位 例如 barrier thread-bound autorelease等等，也包括source的一些标志位
	dispatch_queue_flags_t dqf = 0;
	if (legacy) {
         // OS_dispatch_queue_class 数据类型 ：dispatch_object_vtable_s
		vtable = DISPATCH_VTABLE(queue);
	} else if (dqa->dqa_concurrent) {
         // OS_dispatch_queue_concurrent_class
		vtable = DISPATCH_VTABLE(queue_concurrent);
	} else {
         // OS_dispatch_queue_serial_class
		vtable = DISPATCH_VTABLE(queue_serial);
	}
     // 添加标志位 详细解释 见博客系列基础部分
	switch (dqa->dqa_autorelease_frequency) {
	case DISPATCH_AUTORELEASE_FREQUENCY_NEVER:
        
		dqf |= DQF_AUTORELEASE_NEVER;
		break;
	case DISPATCH_AUTORELEASE_FREQUENCY_WORK_ITEM:
		dqf |= DQF_AUTORELEASE_ALWAYS;
		break;
	}
    // 
	if (label) {
         // 如果 原始 string 是可变的，则重新分配内存，复制内容，变成不可变，否则直接返回
		const char *tmp = _dispatch_strdup_if_mutable(label);
         // 如果新复制的label，添加DQF_LABEL_NEEDS_FREE标志位， 需要 记得释放
		if (tmp != label) {
			dqf |= DQF_LABEL_NEEDS_FREE;
			label = tmp;
		}
	}
    
    // 为队列分配内存 
	dispatch_queue_t dq = _dispatch_alloc(vtable,
			sizeof(struct dispatch_queue_s) - DISPATCH_QUEUE_CACHELINE_PAD);
    // 初始化队列
	_dispatch_queue_init(dq, dqf, dqa->dqa_concurrent ?
			DISPATCH_QUEUE_WIDTH_MAX : 1, dqa->dqa_inactive);

	dq->dq_label = label;

#if HAVE_PTHREAD_WORKQUEUE_QOS
	dq->dq_priority = (dispatch_priority_t)_pthread_qos_class_encode(qos,
			dqa->dqa_relative_priority,
			overcommit == _dispatch_queue_attr_overcommit_enabled ?
			_PTHREAD_PRIORITY_OVERCOMMIT_FLAG : 0);
#endif
	_dispatch_retain(tq);
	if (qos == _DISPATCH_QOS_CLASS_UNSPECIFIED) {
		// legacy way of inherithing the QoS from the target
		_dispatch_queue_priority_inherit_from_target(dq, tq);
	}
	if (!dqa->dqa_inactive) {
		_dispatch_queue_atomic_flags_set(tq, DQF_TARGETED);
	}
	dq->do_targetq = tq;
	_dispatch_object_debug(dq, "%s", __func__);
	return _dispatch_introspection_queue_create(dq);
}
```
```c
// Note to later developers: ensure that any initialization changes are
// made for statically allocated queues (i.e. _dispatch_main_q).
static inline void
_dispatch_queue_init(dispatch_queue_t dq, dispatch_queue_flags_t dqf,
		uint16_t width, bool inactive)
{
    // 设置
    // ((DISPATCH_QUEUE_WIDTH_FULL - (width)) << DISPATCH_QUEUE_WIDTH_SHIFT)
    // (0x8000ull-width) << 37
	uint64_t dq_state = DISPATCH_QUEUE_STATE_INIT_VALUE(width);

	if (inactive) {
		dq_state += DISPATCH_QUEUE_INACTIVE + DISPATCH_QUEUE_NEEDS_ACTIVATION;
		dq->do_ref_cnt++; // rdar://8181908 see _dispatch_queue_resume
	}
	dq->do_next = (struct dispatch_queue_s *)DISPATCH_OBJECT_LISTLESS;
	dqf |= (dispatch_queue_flags_t)width << DQF_WIDTH_SHIFT;
	os_atomic_store2o(dq, dq_atomic_flags, dqf, relaxed);
	dq->dq_state = dq_state;
	dq->dq_override_voucher = DISPATCH_NO_VOUCHER;
	dq->dq_serialnum =
			os_atomic_inc_orig(&_dispatch_queue_serial_numbers, relaxed);
}
```

