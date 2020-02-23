---
title: "iOS 中的锁🔒"
date: 2019-04-21T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC","thread"]
---

# Lock

之前总结了[atomic的安全性问题](https://github.com/FelixScat/Pub/blob/master/posts/atomic.md)，那么在诸如此类的并发使用资源的情况下该如何保证线程安全呢，这篇主要想总结下各种锁的使用

还使用之前电影院卖票的栗子来说明

Theater.h

```objectivec
@interface Theater : NSObject

@property (nonatomic, assign) NSInteger ticketNum;

@end
```

main.m

```objectivec
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        
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
        
        
        NSLog(@"Enter (q) to quit\n");
        char input[100];
        while (scanf("%[^\n]%*c", input)) {
            NSString *str = [NSString stringWithCString:input encoding:NSUTF8StringEncoding];
            if ([str isEqualToString:@"q"]) {
                exit(0);
            }
        }
        
    }
    return 0;
}
```

接下来就用以下不同类型的锁来保障电影院卖票的线程安全吧

## OSSpinLock

OSSpinLock是一种自旋锁，已经被废弃了，因为有可能导致优先级反转的问题

> 新版 iOS 中，系统维护了 5 个不同的线程优先级/QoS: background，utility，default，user-initiated，user-interactive。高优先级线程始终会在低优先级线程前执行，一个线程不会受到比它更低优先级线程的干扰。这种线程调度算法会产生潜在的优先级反转问题，从而破坏了 spin lock。
>
> 具体来说，如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，它会处于 spin lock 的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock。这并不只是理论上的问题，libobjc 已经遇到了很多次这个问题了，于是苹果的工程师停用了 OSSpinLock。

但是既然说到了锁我们还是简单了解下它的用法

```objectivec
static OSSpinLock lock = OS_SPINLOCK_INIT;
OSSpinLockLock(&lock);
theater.ticketNum--;
OSSpinLockUnlock(&lock);
```

## os_unfair_lock

是 OSSpinLock 的替代方案，不过 os_unfair_lock 不是自旋锁而是互斥锁

```objectivec
static os_unfair_lock lock = OS_UNFAIR_LOCK_INIT;
os_unfair_lock_lock(&lock);
theater.ticketNum--;
os_unfair_lock_unlock(&lock);
```

## pthread_mutex

pthread_mutex本质为互斥锁

```objectivec
// 初始化锁
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_DEFAULT);

static pthread_mutex_t mutex;
pthread_mutex_init(&mutex, &attr);

for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {

      pthread_mutex_lock(&mutex);
      theater.ticketNum--;
      pthread_mutex_unlock(&mutex);

      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}


// 使用栅栏函数等待前面任务完成，最后打印卖完票后的总票数
dispatch_barrier_sync(concurrentQueue, ^{
  NSLog(@"%ld", (long)theater.ticketNum);
});

// 销毁
pthread_mutexattr_destroy(&attr);
pthread_mutex_destroy(&mutex);
```

## NSLock

NSLock为pthread_mutex的封装

```objectivec
NSLock *lock = [[NSLock alloc] init];
        
for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {
      [lock lock];
      theater.ticketNum--;
      [lock unlock];

      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}
```

## NSRecursiveLock

pthread_mutex 的封装，能够实现同一线程对资源多次加锁

```objectivec
NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];

// 将总票数分为4部分分别添加到并发队列执行卖票操作
for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {
      [lock lock];
      theater.ticketNum--;
      [lock unlock];

      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}
```

## NSConditionLock

NSConditionLock 也是对pthread_mutex的封装，优点是可以在多处增加加锁的条件，自定义的程度比较高

```objectivec
NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:0];

for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {
      [lock lockWhenCondition:0];
      theater.ticketNum--;
      [lock unlockWithCondition:0];

      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}
```

## @synchronized

本质也是对pthread_mutex的封装

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

## 信号量

dispatch_semaphore_t 是GCD提供的同步机制，其中

dispatch_semaphore_create 用来初始化信号量，

dispatch_semaphore_wait 函数会将信号量的值减1，并且当信号量大小小于0时会一直等待

dispatch_semaphore_signal 函数会将信号量的值加1

```objectivec
dispatch_semaphore_t semaphore_t = dispatch_semaphore_create(1);
        
for (int i = 0; i < 4; i++) {
  dispatch_async(concurrentQueue, ^{
    for (int i = 0; i < 500; i++) {

      dispatch_semaphore_wait(semaphore_t, DISPATCH_TIME_FOREVER);
      theater.ticketNum--;
      dispatch_semaphore_signal(semaphore_t);

      NSLog(@"sold one ticket ,left: %ld", (long)theater.ticketNum);
    }
  });
}
```

