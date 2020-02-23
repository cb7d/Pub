# Crash

## 都有哪些常见的崩溃

大体上，崩溃主要分为两类

- 触发系统规则导致
- 程序逻辑

其中，触发系统规则导致的崩溃主要有以下几类

- 内存警告，通常，在系统内存不足时，系统会向程序发出通知，尝试回收内存，如果还是不够的话将会终止一些后台程序
- 超时，当我们应用程序的appdelegate响应代理方法超时的时候会触发watchDog杀死进程
- 用户强制退出，这里的强制退出不是用户上滑关闭程序，而是类似强制清理内存的操作。

应用程序导致的有以下几种

- SEGV 空指针等操作
- SIGABRT 收到abort信号
- SIGBUS 总线错误，如总线访问异常
- SIGILL 非法指令
- SIGFPE 数学问题
- SIGPIPE 管道问题

常见导致crash的原因：

- unrecognized selector 像对象发送其不能接收的消息
- UI刷新线程
- Timer
- 容器越界
- 通知Notifacation
- KVO

## 如何解决

### 编码规范

通过 runtime 交换方法实现，如Aspect等仓库可以很方便的进行切面处理，但是有些代码的Bug是不适合使用切面处理的，特别是关于支付与用户提交数据相关的事情，崩溃反而是对数据的保护

### RUNTIME

#### unrecognized selector

```objective-c
static void dk_swizzleInstanceMethod(Class cls, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(cls, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzledSelector);
    BOOL didAddMethod = class_addMethod(cls, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
    if (didAddMethod) {
        class_replaceMethod(cls, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

static void dk_swizzleClassMethod(Class cls, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getClassMethod(cls, originalSelector);
    Method swizzledMethod = class_getClassMethod(cls, swizzledSelector);
    BOOL didAddMethod = class_addMethod(cls, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
    if (didAddMethod) {
        class_replaceMethod(cls, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

+ (void)load {
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        dk_swizzleInstanceMethod(self, @selector(forwardingTargetForSelector:), @selector(dk_forwardingTargetForSelector:));
        dk_swizzleClassMethod(self, @selector(forwardingTargetForSelector:), @selector(dk_forwardingTargetForSelector:));
      
    });
}

- (id)dk_forwardingTargetForSelector:(SEL)aSelector {
    
    
    Class dk_class = NSClassFromString(DKProtectClassName);
    if (!dk_class) {
        dk_class = objc_allocateClassPair([NSObject class], DKProtectClassName.UTF8String, 0);
        objc_registerClassPair(dk_class);
    }
    
    NSString *className = NSStringFromClass([self class]);
    if ([className hasPrefix:@"__"]) {
        return [self dk_forwardingTargetForSelector:aSelector];
    }
    
    class_addMethod(dk_class, aSelector, [self dk_safe_implementation:aSelector], [NSStringFromSelector(aSelector) UTF8String]);
    Class Protect = [dk_class class];
    id obj = [[Protect alloc] init];
    return obj;
}

+ (id)dk_forwardingTargetForSelector:(SEL)aSelector {
    
    Class dk_class = NSClassFromString(DKProtectClassName);
    if (!dk_class) {
        dk_class = objc_allocateClassPair([NSObject class], DKProtectClassName.UTF8String, 0);
        objc_registerClassPair(dk_class);
    }
    
    NSString *className = NSStringFromClass([self class]);
    if ([className hasPrefix:@"__"]) {
        return [self dk_forwardingTargetForSelector:aSelector];
    }
    
    class_addMethod(dk_class, aSelector, [self dk_safe_implementation:aSelector], [NSStringFromSelector(aSelector) UTF8String]);
    Class Protect = [dk_class class];
    id obj = [[Protect alloc] init];
    return obj;
}

- (IMP)dk_safe_implementation:(SEL)aSelector
{
    IMP imp = imp_implementationWithBlock(^()                                    {
        NSLog(@"DK Warn :: \nMethod Has Not Been Implement :: %@ %@",NSStringFromClass([self class]) , NSStringFromSelector(aSelector));
    });
    return imp;
}
```

#### UI刷新线程

Hook UI刷新 方法 ，如 `UIView - (void)setNeedsLayout`, hook之后在主线程刷新（有的操作需要提交主线程和主队列，详见SDWebImage）

#### Timer

Timer 循环引用断链，推荐使用NSProxy

#### 容器越界

hook array的index等操作，不建议这样做，应该在开发中避免可能出现的越界操作

#### 通知Notifacation

dealloc中未移出

#### KVO

dealloc中未移出

## 如何解析崩溃

当我们可以拿到真机的时候

打开Xcode -> window -> devices 可以选择 device logs 进行查看，也可以直接调试看看

我们拿不到真机的时候怎么处理呢，这个时候就需要具体打包时候的dysm文件，也叫做符号表，用这个可以解析线上的崩溃信息

## 线上紧急修复

JSPatch等热修复工具