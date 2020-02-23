---
title: "ObjC 中的原子性"
date: 2018-05-11T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC","thread"]
---

# atomic

## 前言

在使用 `@property` 修饰成员变量的时候，会设置一个默认的缺省值 **atomic**

```objectivec
@property (assign) NSInteger ticketNum;
```

以上写法等同于

```objectivec
@property (atomic, assign) NSInteger ticketNum;
```

**atomic** ，顾名思义是原子的，也就是说使用 atomic 修饰能够保证属性的线程安全，那么它真的做到了吗

## 安全性证明

首先定义一个电影院类，向电影院添加一个总票数的属性

```objectivec
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Theater : NSObject

@property (atomic, assign) NSInteger ticketNum;

@end

NS_ASSUME_NONNULL_END
```

接下来我们模拟卖票的过程

```objectivec
Theater *theater = [[Theater alloc] init];
// 定义总票数为2000
theater.ticketNum = 2000;
// 并发队列用于卖票
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.test.example0", DISPATCH_QUEUE_CONCURRENT);

// 将总票数分为4部分分别添加到并发队列执行卖票操作
for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {
      theater.ticketNum--;
      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}

// 使用栅栏函数等待前面任务完成，最后打印卖完票后的总票数
dispatch_barrier_sync(concurrentQueue, ^{
  NSLog(@"%ld", (long)theater.ticketNum);
});
```

你可能会认为，最后打印的一定是0了，但是实际上我们看到

```objectivec
2019-09-21 09:09:47.051759+0800 ObjCSample[8432:89258] sold one ticket ,left: 93
2019-09-21 09:09:47.051763+0800 ObjCSample[8432:89260] sold one ticket ,left: 91
2019-09-21 09:09:47.051764+0800 ObjCSample[8432:89259] sold one ticket ,left: 92
2019-09-21 09:09:47.051769+0800 ObjCSample[8432:89258] sold one ticket ,left: 90
2019-09-21 09:09:47.051771+0800 ObjCSample[8432:89260] sold one ticket ,left: 89
2019-09-21 09:09:47.051911+0800 ObjCSample[8432:89260] sold one ticket ,left: 86
2019-09-21 09:09:47.051936+0800 ObjCSample[8432:89260] sold one ticket ,left: 85
2019-09-21 09:09:47.051780+0800 ObjCSample[8432:89258] sold one ticket ,left: 87
2019-09-21 09:09:47.051965+0800 ObjCSample[8432:89260] sold one ticket ,left: 84
2019-09-21 09:09:47.051784+0800 ObjCSample[8432:89259] sold one ticket ,left: 88
2019-09-21 09:09:47.051974+0800 ObjCSample[8432:89260] sold one ticket ,left: 83
2019-09-21 09:09:47.051983+0800 ObjCSample[8432:89258] sold one ticket ,left: 82
2019-09-21 09:09:47.051992+0800 ObjCSample[8432:89260] sold one ticket ,left: 81
2019-09-21 09:09:47.051998+0800 ObjCSample[8432:89258] sold one ticket ,left: 80
2019-09-21 09:09:47.052003+0800 ObjCSample[8432:89260] sold one ticket ,left: 79
2019-09-21 09:09:47.052005+0800 ObjCSample[8432:89258] sold one ticket ,left: 78
2019-09-21 09:09:47.052009+0800 ObjCSample[8432:89260] sold one ticket ,left: 77
2019-09-21 09:09:47.052011+0800 ObjCSample[8432:89258] sold one ticket ,left: 76
2019-09-21 09:09:47.052015+0800 ObjCSample[8432:89260] sold one ticket ,left: 75
2019-09-21 09:09:47.052125+0800 ObjCSample[8432:89260] sold one ticket ,left: 74
2019-09-21 09:09:47.052141+0800 ObjCSample[8432:89260] sold one ticket ,left: 72
2019-09-21 09:09:47.052138+0800 ObjCSample[8432:89258] sold one ticket ,left: 73
2019-09-21 09:09:47.052297+0800 ObjCSample[8432:88989] 72
```

输出竟然不是0，那么这是为什么呢，我们可以看下苹果提供的[源码](https://opensource.apple.com/tarballs/objc4/)

## 查看源码

找到 objc-accessors.mm 这个文件，可以看到如下方法的定义

```objectivec
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}

void objc_setProperty_atomic(id self, SEL _cmd, id newValue, ptrdiff_t offset)
{
    reallySetProperty(self, _cmd, newValue, offset, true, false, false);
}

static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```

这里在对atomic修饰的属性进行读写的时候，使用了`spinlock_t`进行加锁操作，那么为什么我们上面实践的代码证明了在并发的情况下atomic是无效的呢？

原因就是 `theater.ticketNum--` 这段代码，我们看看它都做了什么

1. 取出 ticketNum
2. ticketNum 的值 减1
3. ticketNum 设置为新值

单独的get或set方法是有保证线程安全的，但是我们在并发的情况下边读边写，这就有可能导致这种情况下出现

> 并发线程中的任务1和任务2同时获取了 ticketNum 的值，比如233，这时任务1向ticketNum写入232，任务2也向ticketNum写入了232，我们期望的递减就没有发生

## 修改方案

最终我们发现，就是由于读写不同步导致atomic不生效，所以我们只需要对 `theater.ticketNum--` 这个操作加锁就好了

```objectivec
for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {
      @synchronized (theater) {
        theater.ticketNum--;
      }
      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}
```

## 其他

在阅读源代码的时候我们可以看到对于原子性修饰的property读写，是使用了spinlock_t加锁的，突然想起来之前看到过的一篇文章，大概意思就是说spinlock_t这个自旋锁在一定情况下会引起死锁。

> 新版 iOS 中，系统维护了 5 个不同的线程优先级/QoS: background，utility，default，user-initiated，user-interactive。高优先级线程始终会在低优先级线程前执行，一个线程不会受到比它更低优先级线程的干扰。这种线程调度算法会产生潜在的优先级反转问题，从而破坏了 spin lock。
>
> 具体来说，如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，它会处于 spin lock 的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock。这并不只是理论上的问题，libobjc 已经遇到了很多次这个问题了，于是苹果的工程师停用了 OSSpinLock。

可是我们在阅读源码的时候看到明明用的还是spinlock，不是说不安全了吗，

objc-os.h

```cpp
using spinlock_t = mutex_tt<LOCKDEBUG>;
#include "objc-lockdebug.h"

template <bool Debug>
class mutex_tt : nocopy_t {
    os_unfair_lock mLock;
 public:
    constexpr mutex_tt() : mLock(OS_UNFAIR_LOCK_INIT) {
        lockdebug_remember_mutex(this);
    }

    constexpr mutex_tt(const fork_unsafe_lock_t unsafe) : mLock(OS_UNFAIR_LOCK_INIT) { }

    void lock() {
        lockdebug_mutex_lock(this);

        os_unfair_lock_lock_with_options_inline
            (&mLock, OS_UNFAIR_LOCK_DATA_SYNCHRONIZATION);
    }

    void unlock() {
        lockdebug_mutex_unlock(this);

        os_unfair_lock_unlock_inline(&mLock);
    }

    void forceReset() {
        lockdebug_mutex_unlock(this);

        bzero(&mLock, sizeof(mLock));
        mLock = os_unfair_lock OS_UNFAIR_LOCK_INIT;
    }

    void assertLocked() {
        lockdebug_mutex_assert_locked(this);
    }

    void assertUnlocked() {
        lockdebug_mutex_assert_unlocked(this);
    }


    // Address-ordered lock discipline for a pair of locks.

    static void lockTwo(mutex_tt *lock1, mutex_tt *lock2) {
        if (lock1 < lock2) {
            lock1->lock();
            lock2->lock();
        } else {
            lock2->lock();
            if (lock2 != lock1) lock1->lock(); 
        }
    }

    static void unlockTwo(mutex_tt *lock1, mutex_tt *lock2) {
        lock1->unlock();
        if (lock2 != lock1) lock2->unlock();
    }

    // Scoped lock and unlock
    class locker : nocopy_t {
        mutex_tt& lock;
    public:
        locker(mutex_tt& newLock) 
            : lock(newLock) { lock.lock(); }
        ~locker() { lock.unlock(); }
    };

    // Either scoped lock and unlock, or NOP.
    class conditional_locker : nocopy_t {
        mutex_tt& lock;
        bool didLock;
    public:
        conditional_locker(mutex_tt& newLock, bool shouldLock)
            : lock(newLock), didLock(shouldLock)
        {
            if (shouldLock) lock.lock();
        }
        ~conditional_locker() { if (didLock) lock.unlock(); }
    };
};
```

无语，内部偷偷换成 os_unfair_lock ，一个改进版的互斥锁， 外面还不改名字，真的骚

## 总结

总之用atomic修饰property并没有什么用处，除非你的属性是只读的，不然在读写操作的时候我们还是要自己去加锁来保证线程安全，还有一点非常重要，就是不确定的东西多去看看源代码，不然还要纠结半天。



