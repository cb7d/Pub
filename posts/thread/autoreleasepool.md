# Autoreleasepool

我们可以看到我们项目中的main.m中声明了这个关键字，autoreleasepool，它是干什么用的呢

```objectivec
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

首先，我们知道它是用来释放对象的，在arc下系统会为我们管理内存，那么它是如何工作的呢

```objectivec
- (void)someBussiness {
    
    
    NSString *str = @"123";
    
}
```

我们在这里创建字符串后面打个断点，在到达断点时我们使用lldb为该变量添加观察

```shell
(lldb) watchpoint set variable str
Watchpoint created: Watchpoint 1: addr = 0x70000b5d0c58 size = 8 state = enabled type = w
    declare @ '/X/demo/ViewController.m:42'
    watchpoint spec = 'str'
    new value: 0x000000010f9ad070
(lldb) 
```

接着点击继续，会在后续的断点中看到

```assembly
CoreFoundation`objc_autoreleasePoolPop:
->  0x110d7e614 <+0>: jmpq   *0x1ce4b6(%rip)           ; (void *)0x0000000110295b6e: objc_autoreleasePoolPop
```

这里调用了pop方法，看到这里我们可以猜测这个autoreleasePool应该是个栈的结构，使用push和pop来帮助我们自动释放内存，下面从源码的角度去分析其原理

首先，我们使用clang重写main.m

```cpp
int main(int argc, char * argv[]) {
    @autoreleasepool {
        
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

转化后

```sh
clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk demo/main.m
```

当前目录下会生成一个main.c（省略部分代码）

```c
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```

可以看到`@autoreleasepool`被转化成了这个东西`__AtAutoreleasePool __autoreleasepool`，继续搜索我们会看到这个结构体

```cpp
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

可以看到，初始化的时候会调用objc_autoreleasePoolPush这个方法，析构的时候会调用objc_autoreleasePoolPop，其实就相当于在 `@autoreleasepool` 包裹的中间代码中上面插入一个push，下面插入一个pop

看到这里还是不够，我们可以查看[源代码](https://opensource.apple.com/tarballs/objc4/)

- objc_autoreleasePoolPush
- objc_autoreleasePoolPop

在查看源码时可以看到这个定义

```cpp
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

可以看到这里都是调用了 AutoreleasePoolPage 的方法

## AutoreleasePoolPage

这里我们先看下Page的大致结构如下

```cpp
class AutoreleasePoolPage 
{
  	// 用来校验page的完整性
    magic_t const magic;
    id *next;
  	// 保存当前线程
    pthread_t const thread;
  	// 指向上一页的指针
    AutoreleasePoolPage * const parent;
  	// 指向下一页的指针
    AutoreleasePoolPage *child;
  	// 深度
    uint32_t const depth;
    uint32_t hiwat;
}
```

可以看出，自动释放池的结构是一个双向链表

```cpp
#define I386_PGBYTES            4096            /* bytes per 80386 page */
#define PAGE_SIZE               I386_PGBYTES
```

每页占用的最大内存为 4096 字节

## objc_autoreleasePoolPush

看完大致结构继续查看push操作

```cpp
static inline void *push() 
{
  id *dest;
  dest = autoreleaseFast(POOL_BOUNDARY);
  assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
  return dest;
}
```

又会进入以下方法

```cpp
static inline id *autoreleaseFast(id obj)
{
  // 获取hotPage
  AutoreleasePoolPage *page = hotPage();
  // 获取到page并且page没有满
  if (page && !page->full()) {
    // page直接添加新对象
    return page->add(obj);
  } else if (page) {
    // page满了
    return autoreleaseFullPage(obj, page);
  } else {
    // 没有获取到page
    return autoreleaseNoPage(obj);
  }
}
```

下面就会分成三种情况

获取到hotPage且没有满

```cpp
id *add(id obj)
{
  assert(!full());
  unprotect();
  // 压栈
  id *ret = next;  // faster than `return next-1` because of aliasing
  *next++ = obj;
  protect();
  return ret;
}
```

hotPage已经满员

```cpp
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
  assert(page == hotPage());
  assert(page->full()  ||  DebugPoolAllocation);
	// 顺着链表向下查找是否有未满的page，没有则创建新的page
  do {
    if (page->child) page = page->child;
    else page = new AutoreleasePoolPage(page);
  } while (page->full());

  setHotPage(page);
  return page->add(obj);
}
```

未获取到hotPage

```cpp
id *autoreleaseNoPage(id obj)
{
  // 进入此方法有两种可能，一是自动释放池没有执行push操作，二是自动释放池为空
  assert(!hotPage());

  bool pushExtraBoundary = false;
  if (haveEmptyPoolPlaceholder()) {
    // We are pushing a second pool over the empty placeholder pool
    // or pushing the first object into the empty placeholder pool.
    // Before doing that, push a pool boundary on behalf of the pool 
    // that is currently represented by the empty placeholder.
    pushExtraBoundary = true;
  }
  else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
    // We are pushing an object with no pool in place, 
    // and no-pool debugging was requested by environment.
    _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                 "autoreleased with no pool in place - "
                 "just leaking - break on "
                 "objc_autoreleaseNoPool() to debug", 
                 pthread_self(), (void*)obj, object_getClassName(obj));
    objc_autoreleaseNoPool(obj);
    return nil;
  }
  else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
    // We are pushing a pool with no pool in place,
    // and alloc-per-pool debugging was not requested.
    // Install and return the empty pool placeholder.
    return setEmptyPoolPlaceholder();
  }

  // We are pushing an object or a non-placeholder'd pool.

  // 初始化新的一页并设诶hotPage
  AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
  setHotPage(page);

  // Push a boundary on behalf of the previously-placeholder'd pool.
  if (pushExtraBoundary) {
    page->add(POOL_BOUNDARY);
  }

  // Push the requested object or pool.
  return page->add(obj);
}
```

## objc_autoreleasePoolPop

下面我们回过头看看pop方法

```cpp
static inline void pop(void *token) 
{
  AutoreleasePoolPage *page;
  id *stop;
	
  // 清理 placeholder pool
  if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
    if (hotPage()) {
      pop(coldPage()->begin());
    } else {
      setHotPage(nil);
    }
    return;
  }
	// 获取当前传入token所在的page
  page = pageForPointer(token);
  stop = (id *)token;
  if (*stop != POOL_BOUNDARY) {
    if (stop == page->begin()  &&  !page->parent) {
      // Start of coldest page may correctly not be POOL_BOUNDARY:
      // 1. top-level pool is popped, leaving the cold page in place
      // 2. an object is autoreleased with no pool
    } else {
      // Error. For bincompat purposes this is not 
      // fatal in executables built with old SDKs.
      return badPop(token);
    }
  }

  if (PrintPoolHiwat) printHiwat();

  // 循环next清理内容，直到stop
  page->releaseUntil(stop);

  // 清理空的链表下级
  if (DebugPoolAllocation  &&  page->empty()) {
    // special case: delete everything during page-per-pool debugging
    AutoreleasePoolPage *parent = page->parent;
    page->kill();
    setHotPage(parent);
  } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
    // special case: delete everything for pop(top) 
    // when debugging missing autorelease pools
    page->kill();
    setHotPage(nil);
  } 
  else if (page->child) {
    // hysteresis: keep one empty child if page is more than half full
    if (page->lessThanHalfFull()) {
      page->child->kill();
    }
    else if (page->child->child) {
      page->child->child->kill();
    }
  }
}
```

其中 releaseUntil 方法是这样的

```cpp
void releaseUntil(id *stop) 
{
  // Not recursive: we don't want to blow out the stack 
  // if a thread accumulates a stupendous amount of garbage

  while (this->next != stop) {
    // Restart from hotPage() every time, in case -release 
    // autoreleased more objects
    AutoreleasePoolPage *page = hotPage();

    // fixme I think this `while` can be `if`, but I can't prove it
    while (page->empty()) {
      page = page->parent;
      setHotPage(page);
    }

    page->unprotect();
    id obj = *--page->next;
    memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
    page->protect();

    if (obj != POOL_BOUNDARY) {
      // release page 中的对象
      objc_release(obj);
    }
  }

  setHotPage(this);

  #if DEBUG
  // we expect any children to be completely empty
  for (AutoreleasePoolPage *page = child; page; page = page->child) {
    assert(page->empty());
  }
  #endif
}

```

那么，对象是如何加入自动释放池的呢

对象在发送autorelease的时候

```cpp
- (id)autorelease {
    return ((id)self)->rootAutorelease();
}
```

最终会调用到这个方法

```cpp
static inline id *autoreleaseFast(id obj)
{
  AutoreleasePoolPage *page = hotPage();
  if (page && !page->full()) {
    return page->add(obj);
  } else if (page) {
    return autoreleaseFullPage(obj, page);
  } else {
    return autoreleaseNoPage(obj);
  }
}
```

到了这里，关于自动释放池我们已经知道了大概运行的原理，总结一下

在使用@autoreleasepool的时候：

1. 调用 objc_autoreleasePoolPush 创建新page
2. 对象通过发送 autorelease 消息加入自动释放池
3. 自动释放池context结束调用objc_autoreleasePoolPop，向pool中对象发送release消息

## 参考

[https://draveness.me/autoreleasepool](https://draveness.me/autoreleasepool)

[https://opensource.apple.com/tarballs/objc4/](https://opensource.apple.com/tarballs/objc4/)

[https://juejin.im/post/5a687e356fb9a01c94060620](https://juejin.im/post/5a687e356fb9a01c94060620)