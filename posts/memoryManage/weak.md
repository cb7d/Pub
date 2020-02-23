---
title: "ObjC 的 weak 修饰"
date: 2019-07-19T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC"]
---

# Weak

我们在代码中可以找到很多地方使用了weak来修饰成员变量，比如

```objectivec
@property (nonatomic, weak, nullable) id <UITableViewDelegate> delegate;
```

## Weak的作用

> weak 用于修饰的对象被赋值不会引起retain，销毁也不会release，weak指针指向的对象一旦被释放，weak的指针将会被置为nil

大部分情况下我们的代码中都会存在着各种持有其他对象的情况，而被持有的对象可能由于某些需求需要访问原对象，那么就会导致互相持有对方，在ARC下，俩个对象互相持有将导致内存一直无法释放，因为每个对象在自动释放池中的retainCount都大于0。由于weak的特性，这个时候就轮到他出场了

## 什么时候应该用Weak

- 通常在使用代理模式的时候应该使用weak来避免循环引用
- 在使用block时需要引用block外部的变量的时候可以用weak来断链
- 需要对象释放是自动置为nil，防止野指针错误

> 这里有个小技巧，只要你的类要完成的功能是需要**借用**的，就用weak

## Weak实现的机制

我们现在只知道在weak对象销毁后会置为nil，接下来我们从源码角度看看weak到底是如何实现的

```objectivec
NSObject *one = [[NSObject alloc] init];

__weak typeof(one) oneWeak = one;
```

在代码中单步断点，我们可以看到进入了这个方法

```assembly
ObjCSample`objc_initWeak:
->  0x100000e7e <+0>: jmpq   *0x1cc(%rip)              ; (void *)0x0000000100000ed0
```

看到这个方法就好办了，我们掏出源码瞅瞅这个方法做了啥

NSObject.mm

```cpp
/** 
 * Initialize a fresh weak pointer to some object location. 
 * It would be used for code like: 
 *
 * (The nil case) 
 * __weak id weakPtr;
 * (The non-nil case) 
 * NSObject *o = ...;
 * __weak id weakPtr = o;
 * 
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 *
 * @param location Address of __weak ptr. 
 * @param newObj Object ptr. 
 */
id
objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```

可以看到这里是个外层方法，仅判断了对象是不是nil，下面会进入

```cpp
// 这里用于更新 weak 变量
// 如果 HaveOld 为 true，说明变量已经有存在于weak表中的值了，需要清理
// 如果 HaveNew 为 true，说明变量需要被赋值
// 如果 CrashIfDeallocating 为 true，说明 newObj 已经释放或者 newObj 不支持弱引用
enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    // 声明新值和旧值所在的SideTable
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    // 获取旧对象和旧对象的SideTable
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
		// 获取新对象所在的SideTable
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
		// 对两张表加锁，防止多线程资源抢夺
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
		
    // 若旧值和从指针取出的值不一样则认为已经被其他线程修改，进行重试
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }
            
    if (haveNew  &&  newObj) {
      	// 获取isa指针
        Class cls = newObj->getIsa();
      	// 如果isa非空并且isa没有被初始化，初始化isa并重试
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // 清理旧值
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // 新值赋值
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // 在引用计数表中标记
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // 改变location指向新的对象
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

可以看到取对象的代码是 `&SideTables()[oldObj]` 那么这个东西里面是什么呢

```cpp
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```

其中：

- reinterpret_cast 为 c++ 的类型转换方法，简单来说就是使用 `StripedMap<SideTable>*` 的方式来读写 SideTableBuf 的值
- `StripedMap<T>` 是一个模版类，根据传递的实际参数决定其中array成员存储的元素类型，能通过对象的地址计算Hash值，通过该值找到对应的value

下面我们接着看 SideTable

```cpp
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<HaveOld, HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<HaveOld, HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

SideTable 我觉得翻译成 **小边桌** 比较合适，就像角落里的桌子，没多少人但是也必须有。我们看它的数据结构

- spinlock_t slock 用来保障多线程安全（注：别被名字骗了，早就不是自旋锁了）
- RefcountMap refcnts 管理对象的引用计数
- weak_table_t weak_table 真正的weak表

下面看weak表是个什么结构

```cpp
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
/**
 * 全局弱引用的表，将对象的id作为key
 * weak_entry_t 作为value
 */
struct weak_table_t {
  	// 存储所有指向指定对象的 weak 指针，也就是最终包装weak对象的地方
    weak_entry_t *weak_entries;
  	// 容量
    size_t    num_entries;
  	// 掩码
    uintptr_t mask;
  	// Hash key 最大偏移量
    uintptr_t max_hash_displacement;
};
```

再来看下 weak_entry_t 

```cpp
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
  	// 内存中的weak对象，DisguisedPtr是为了防止被误报为内存泄漏
    DisguisedPtr<objc_object> referent;
    union {
        struct {
          	// 所有指向weak对象的变量
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```

### 小总结一下

综上可以我们可以得出一下结论

- weak_table_t 是一个全局的弱引用表，负责存储全局的弱引用数据
- weak_table_t 中的 weak_entries 负责维护指向每一个对象的所有弱引用 Hash 表

下面看看注册weak表的方法

```cpp
/** 
 * Registers a new (object, weak pointer) pair. Creates a new weak
 * object entry if it does not exist.
 * 
 * @param weak_table The global weak table.
 * @param referent The object pointed to by the weak reference.
 * @param referrer The weak pointer address.
 */
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
  	// 对象的值，也就是weak表中的key
    objc_object *referent = (objc_object *)referent_id;
  	// 对象的地址（指针）
    objc_object **referrer = (objc_object **)referrer_id;
		// 如果对象为空或者该对象使用了 TaggedPointer 技术则直接返回原对象（注：关于 TaggedPointer 本文暂且不表，以后会专门写一篇）
    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // 对象是否有效
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }
		// 对象 dealloc
    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // 判断对象是否有关联的弱引用表
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
      	// 添加关联
        append_referrer(entry, referrer);
    } 
    else {
      	// 创建weak实体
        weak_entry_t new_entry(referent, referrer);
      	// 表满扩容
        weak_grow_maybe(weak_table);
      	// 插值
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
```

接下来我们看看旧对象的解除操作

```cpp
/** 
 * Unregister an already-registered weak reference.
 * This is used when referrer's storage is about to go away, but referent
 * isn't dead yet. (Otherwise, zeroing referrer later would be a
 * bad memory access.)
 * Does nothing if referent/referrer is not a currently active weak reference.
 * Does not zero referrer.
 * 
 * FIXME currently requires old referent value to be passed in (lame)
 * FIXME unregistration should be automatic if referrer is collected
 * 
 * @param weak_table The global weak table.
 * @param referent The object.
 * @param referrer The weak reference.
 */
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
  	// 旧对象的值，也就是weak表中的key
    objc_object *referent = (objc_object *)referent_id;
  	// 旧对象的地址（指针）
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;
		// 对象为nil直接返回
    if (!referent) return;
		// 根据weak表和旧对象的值查找weak_entry
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
      	// 删除引用关联
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }
				// 如果该对象的弱引用列表为空则从table中移除
        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```

我们在使用weak修饰对象的时候显然是不会调用这个方法的，那么啥情况下会执行呢

```objectivec
- (void)testMethod {
    __weak typeof(self) weakSelf = self;
    ...
    ...
}
```

我们在这里声明了 weakSelf 这个局部变量，在方法作用结束之后局部变量将被释放，这个时候就会调用解除操作方法把 weakSelf 从当前对象的弱引用列表中移除。

### 在对象释放的时候weak表又做了啥

我们可以一层一层从NSObject的dealloc开始追溯

NSObject.mm

```cpp
- (void)dealloc {
    _objc_rootDealloc(self);
}
```

objc-object.h

```cpp
inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?

    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```

最终到达这个方法

```cpp
/** 
 * Called by dealloc; nils out all weak pointers that point to the 
 * provided object so that they can no longer be used.
 * 
 * @param weak_table 
 * @param referent The object being deallocated. 
 */
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
  	// 正在dealloc的对象
    objc_object *referent = (objc_object *)referent_id;
		// 根据weak表和该对象的值找到weak对应的entry
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    // 删除引用关联
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    // 删除对象关联的weak_entry
    weak_entry_remove(weak_table, entry);
}
```

## 总结

到这里整个weak的实现思路都完成了，我们总结下初始化和释放的过程吧

### 初始化过程

1. 获取 SideTable 
2. 获取 weak_table_t（key为赋值对象地址的hash）
3. 生成 weak_entry_t （主要作用包装被weak修饰的指针变量地址）

### 释放过程

1. 获取 SideTable
2. 获取 weak_table_t
3. 获取 weak_entry_t
4. 销毁所有 weak_entry_t 指向对象的指针
5. 销毁 weak_entry_t

