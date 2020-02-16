# Grand Central Dispatch

这篇文主要想总结下 `GCD` 在swift中的使用，[文中示例代码](https://github.com/FelixScat/demo_GCD)

[ObjC版本](https://github.com/FelixScat/demo_GCD/blob/master/ObjCGCD/README.md)

## 基本概念

### 进程

进程指在系统中能独立运行并作为资源分配的基本单位，它是由一组机器指令、数据和堆栈等组成的，是一个能独立运行的活动实体

### 线程

线程是进程的基本执行单元，一个进程（程序）的所有任务都在线程中执行。

### 队列

队列，又称为伫列（queue），是先进先出（FIFO, First-In-First-Out）的线性表。在具体应用中通常用链表或者数组来实现。队列只允许在后端（称为rear）进行插入操作，在前端（称为front）进行删除操作。队列的操作方式和堆栈类似，唯一的区别在于队列只允许新数据在后端进行添加。

### 同步/异步

可以这么理解：

假如你要做两件事 ， 烧水 、 刷牙

- 同步 ：你烧水 ， 等水烧开了你再去刷牙
- 异步 ：你烧水 ，不等水烧开就去刷牙了 ， 水烧开了会发出声音告诉你（callback） ， 然后你再处理水烧开之后的事情

**只要你是个正常人 ， 都会选择第二种 ，当然也有特殊情况 ，你喜欢用热水刷牙**

### 并发

指两个或多个事件在同一时间间隔内发生。可以在某条线程和其他线程之间反复多次进行上下文切换，看上去就好像一个CPU能够并且执行多个线程一样。其实是伪异步。

### 线程队列中并行/串行

串行队列：串行队列的特点是队列内的线程是一个一个执行，直到结束。并行队列：并行队列的特点是队列中所有线程的执行结束时必须是一块的，队列中其他线程执行完毕后，会阻塞当前线程等待队列中其他线程执行，然后一块执行完毕。

***

## 开始

下面我们就用刷牙与烧水来举例，首先clone工程，[本文工程Demo](https://github.com/FelixScat/demo_GCD)

```sh
git clone https://github.com/FelixScat/demo_GCD.git
cd swiftGCD
swift package generate-xcodeproj
xed ./
```

打开`main.swift`先声明两个事件

```swift
/// 烧水
let boiledWater = {
    print("开始烧水: \(Thread.current)")
    sleep(3)
    print("水烧好啦")
}

/// 刷牙
let brushTeeth = {
    print("开始刷牙:\(Thread.current)")
    sleep(5)
    print("牙刷完啦")
}
```

### 队列

#### 先声明两个队列

```swift
/// 串行队列
let serialQueue = DispatchQueue(label: "top.felixplus.k.serial")
/// 并行队列
let concurrentQueue = DispatchQueue(label: "top.felixplus.k.concurrent", attributes: .concurrent)
```

其中串行队列就表示队列中的人物会依次执行，而并行队列中的人物将会同时并发执行

### 同步任务

```swift
DispatchQueue.global().sync {
    boiledWater()
    brushTeeth()
}
```

串行就会一个任务接着一个任务执行，最终输入如下

```
开始烧水: <NSThread: 0x100603730>{number = 1, name = main}
水烧好啦
开始刷牙:<NSThread: 0x100603730>{number = 1, name = main}
牙刷完啦
```

使用sync是没有开辟新线程的能力的，同时，同步任务执行的线程必然为sync代码执行上下文的线程

可以看到log中的输出是在mainthread。

### 异步任务

```swift
DispatchQueue.global().async(execute: boiledWater)
DispatchQueue.global().async(execute: brushTeeth)
```

并行任务几乎会同时开始，同时进行，下面是输出

```
开始刷牙:<NSThread: 0x100700000>{number = 2, name = (null)}
开始烧水: <NSThread: 0x102805500>{number = 3, name = (null)}
水烧好啦
牙刷完啦
```

异步任务也不一定就一定会开启新的线程，具体的操作会由GCD内部负责处理，可以尝试以下测试代码

```swift
(1...1000).forEach { (i) in
    DispatchQueue.global().async(execute: boiledWater)
    DispatchQueue.global().async(execute: brushTeeth)
}
```

### DispatchWorkItem

上面的两个任务其本质就是两个函数，不过在GCD中还有更加方便的写法

```swift
/// 烧水
let boiledWater = DispatchWorkItem{
    print("开始烧水: \(Thread.current)")
    sleep(3)
    print("水烧好啦")
}

/// 刷牙
let brushTeeth = DispatchWorkItem{
    print("开始刷牙:\(Thread.current)")
    sleep(5)
    print("牙刷完啦")
}
```

我们依然可以用之前的调用方式来执行任务

```swift
DispatchQueue.global().async(execute: boiledWater)
DispatchQueue.global().async(execute: brushTeeth)
```

除此之外还可以这样执行任务

```
boiledWater.perform()
brushTeeth.perform()
```

这就相当于使用`sync`方法在当前的thread同步添加任务

还能够取消任务

```
DispatchQueue.global().async {
    boiledWater.perform()
    brushTeeth.perform()
}
boiledWater.cancel()
```

输出如下

```
开始刷牙:<NSThread: 0x100600240>{number = 2, name = (null)}
牙刷完啦
```

### 优先级QOS

队列在执行的时候有优先级的区别，更高的优先级会得到更优先调用顺序

- User Interactive： 和用户交互相关，比如动画等等优先级最高。比如用户连续拖拽的计算
- User Initiated： 需要立刻的结果，比如push一个ViewController之前的数据计算
- Utility： 可以执行很长时间，再通知用户结果。比如下载一个文件，给用户下载进度。
- Background： 用户不可见，比如在后台存储大量数据

```swift
/// 串行队列
let serialQueue = DispatchQueue(label: "top.felixplus.k.serial", qos: .default)
/// 并行队列
let concurrentQueue = DispatchQueue(label: "top.felixplus.k.concurrent", qos: .default, attributes: .concurrent)
```

### 死锁

```swift
DispatchQueue.main.sync {
    boiledWater.perform()
}
```

这段代码一定会造成死锁，那么产生的原因是什么呢

可以先把上面这一整段代码想象成一个任务块(block)，当前是主队列，主队列是串行队列，主队列只有当前的block执行完毕才会执行下一个，上面代码中在一个还没有结束的block中增加了一个任务，并且要阻塞当前的队列优先去执行，所以最终导致了队列阻塞，记住一点，类似上述代码导致死锁的原因并不是线程阻塞，而是队列阻塞

解决方法有很多，根据上述我们所说的队列问题，其实很简单，我们使用自己新建的队列执行就可以避免

```swift
serialQueue.sync {
    boiledWater.perform()
}
```

这样代码依然会在mainthread执行，并且不会导致主队列的阻塞

### 信号量

信号量可以在很多场景下使用，初始化信号量需要一个数值，信号量通过wait来将这个数值减1，通过signal方法来将这个数值加1，当信号量的值小于0的时候将会一直等待

举个例子，通过信号量我们可以限制一个并行队列中同时运行的任务数量

```swift
let signal =  DispatchSemaphore(value: 2)

signal.wait()
DispatchQueue.global().async {
    boiledWater.perform()
    signal.signal()
}

signal.wait()
DispatchQueue.global().async {
    brushTeeth.perform()
    signal.signal()
}

signal.wait()
DispatchQueue.global().async {
    boiledWater.perform()
    signal.signal()
}

signal.wait()
DispatchQueue.global().async {
    brushTeeth.perform()
    signal.signal()
}
```

再举个例子，我们可以将一些异步的任务转为同步执行（水烧好再刷牙）

```swift
let signal =  DispatchSemaphore(value: 0)
concurrentQueue.async {
    boiledWater.perform()
    signal.signal()
}

signal.wait()

concurrentQueue.async {
    brushTeeth.perform()
}
```

### Group

任务组可以用来管理任意的任务，不管他们是来自相同队列还是不同队列，下面是使用

```swift
let group = DispatchGroup()

concurrentQueue.async(group: group, execute: boiledWater)
concurrentQueue.async(group: group, execute: brushTeeth)
```

也可以使用group的enter和leave方法

```swift
group.enter()
concurrentQueue.async {
    boiledWater.perform()
    group.leave()
}

group.enter()
serialQueue.sync {
    brushTeeth.perform()
    group.leave()
}
```

在任务完成时发出通知

```swift
group.notify(queue: concurrentQueue) {
    print("All done")
}
```

阻塞当前线程直到任务全部完成

```swift
group.wait()
print("All done")
```

### 栅栏方法

栅栏方法顾名思义，会把当前的任务前后加上**围栏** 

- 执行当前任务需要队列中前面全部的任务执行完毕
- 需要当前任务执行完毕才会执行后面的函数

```swift
/// 烧水
let boiledWaterWithBarrier = DispatchWorkItem(qos: .default, flags: .barrier) {
    print("开始烧水: \(Thread.current)")
    sleep(3)
    print("水烧好啦")
}

/// 刷牙
let brushTeethWithBarrier = DispatchWorkItem(qos: .default, flags: .barrier) {
    print("开始刷牙:\(Thread.current)")
    sleep(5)
    print("牙刷完啦")
}

concurrentQueue.async(execute: boiledWaterWithBarrier)
concurrentQueue.async(execute: brushTeethWithBarrier)
```

### 迭代

DispatchQueue为我们提供了一种更加方便的方法来同时执行多个迭代次数的相同任务

当我们有大量细小的重复性的工作的时候可以这么用

比如我们要找到0到100000中所有能被17整除的数字：

```swift
let list = Array(0...100000)
var result = [Int]()


concurrentQueue.async {
    
    
    DispatchQueue.concurrentPerform(iterations: list.count) { (i) in
        if (i % 17 == 0) {
            serialQueue.sync {
                result.append(list[i])
            }
        }
    }
    serialQueue.sync(execute: {
        print(result)
    })
}
```



## 参考

- [https://developer.apple.com/documentation/dispatch](https://developer.apple.com/documentation/dispatch)
- [https://stackoverflow.com/questions/23856230/what-is-the-difference-between-gcd-main-queue-and-the-main-thread](https://stackoverflow.com/questions/23856230/what-is-the-difference-between-gcd-main-queue-and-the-main-thread)
- [https://developer.apple.com/documentation/dispatch/dispatchqueue/2016088-concurrentperform](https://developer.apple.com/documentation/dispatch/dispatchqueue/2016088-concurrentperform)
- [https://juejin.im/post/5acaea17f265da239a601a01#heading-14](https://juejin.im/post/5acaea17f265da239a601a01#heading-14)
- [https://medium.com/@vikasdalvi.29/multitasking-in-ios-using-gcd-b931885a719e](https://medium.com/@vikasdalvi.29/multitasking-in-ios-using-gcd-b931885a719e)










