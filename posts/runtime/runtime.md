# Runtime

我们看下[苹果官方文档](https://developer.apple.com/documentation/objectivec/objective-c_runtime)对runtime的定义

> The Objective-C runtime is a runtime library that provides support for the dynamic properties of the Objective-C language, and as such is linked to by all Objective-C apps. Objective-C runtime library support functions are implemented in the shared library found at /usr/lib/libobjc.A.dylib.

译文如下

> Objective-C运行时是一个运行时库，它提供对Objective-C语言的动态属性的支持，因此被所有Objective-C应用程序链接。 Objective-C运行时库支持函数在/usr/lib/libobjc.A.dylib中的共享库中实现。

在Objective-C中，消息直到运行时才绑定到方法实现。编译器将把方法调用转化为消息发送

这也是**Objective-C**被称为**动态语言**的原因

例如如下代码

```objectivec
[receiver message]
```

将会被转化为这种调用方式

```objectivec
objc_msgSend(receiver, selector)
```

在消息需要绑定参数的时候会转化如下

```objectivec
objc_msgSend(receiver, selector, arg1, arg2, ...)
```

那么抓花为发送消息之后都做了什么呢? 

```objectivec
[receiver message]
```

1. 通过receiver的 isa 指针 查找它的 Class
2. 查找 Class 下的 methodLists
3. 如果 methodLists 没有相应的方法则递归查找 superClass 的 methodLists
4. 如果在 methodLists 里面找到了 对应的 message 则 获取实现指针 imp 并执行
5. 发送方法返回值

这里我们发现还缺少了一种情况，那就是递归在父类的methodlist里面也没有找到对应的实现，这个时候就会报错 `unrecognized selector send to instance X` 

## 消息转发

Runtime 为这种可能提供了最后的机会，就是触发消息转发流程

1. resolveClassMethod/resolveInstanceMethod ，向对象发送其无法识别的消息后会触发，在这里可以动态添加方法的实现
2. forwardingTargetForSelector ，快速转发，可以把对应的消息发送给其他对象
3. methodSignatureForSelector ，对方法进行签名(为完整的消息转发做准备)
4. forwardInvocation ，进行完整的消息转发(包括修改实际方法，对象等)
5. doesNotRecognizeSelector , 最后如果还未执行方法，就会抛出错误

Show Me The Code:

动态添加方法：

```objectivec
#import "AViewController.h"
#import <objc/runtime.h>

@interface AViewController ()

@end

@implementation AViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelector:@selector(speak)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    
    if (sel == @selector(speak)) {
        class_addMethod([self class], sel, (IMP)fakeSpeak, "v@:");
        // 关于最后一个参数可以看https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html
        return true;
    }
    return [super resolveInstanceMethod:sel];
}

void fakeSpeak(id target, SEL _cmd){
    
    NSLog(@"method added");
}

@end
```

快速转发

```objectivec
#import "AViewController.h"

#import <objc/runtime.h>

@interface AViewController ()

@end

@implementation AViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelector:@selector(speak)];
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    if (aSelector == @selector(speak)) {
        
        return [XXXX new];
    }
    
    return nil;
}

@end
```

完整转发

```objectivec
#import "AViewController.h"

#import <objc/runtime.h>

@interface AViewController ()

@end

@implementation AViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelector:@selector(speak)];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    
    if (aSelector == @selector(speak)) {
        
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    [anInvocation setSelector:@selector(otherMethod)];
    [anInvocation invokeWithTarget:self];
}

- (void)otherMethod{
    NSLog(@"%s",__func__);
}

@end
```

对消息转发的流程有了一些基本概念以后我们就可以稍微深入看看方法交换这个理念了。

#### 方法交换

有的时候我们可能会面对一些需求，比如在每个页面中统一都做的一些处理，像访问埋点等逻辑，如果一个一个去改写的话十分麻烦，用继承的方式去做慢慢会产生各种耦合的情况，这里，我们可以使用方法交换的方式去统一添加处理。

比如我们需要在每一个 ViewController viewDidLoad 的方法中输出一个log
先创建一个 category


```objectivec
#import "UIViewController+Log.h"
#import <objc/runtime.h>

@implementation UIViewController (Log)

static void AGExchangeMethod(Class cls, SEL originSelector, SEL newSelector) {
    
    Method originMethod = class_getInstanceMethod(cls, originSelector);
    Method newMethod = class_getInstanceMethod(cls, newSelector);
    
    //    method_exchangeImplementations(newMethod, originMethod);
    
    BOOL addMethod = class_addMethod(cls, originSelector, method_getImplementation(newMethod), method_getTypeEncoding(newMethod));
    
    if (addMethod) {
        
        class_replaceMethod(cls, newSelector, method_getImplementation(originMethod), method_getTypeEncoding(originMethod));
        
    }else {
        
        method_exchangeImplementations(newMethod, originMethod);
    }
}

+ (void)load {
    static dispatch_once_t once;
    dispatch_once(&once, ^{
       
        AGExchangeMethod([self class], @selector(viewDidLoad), @selector(Logging));
    });
}

- (void)Logging{
    
    NSLog(@"%s",__func__);
    
    [self Logging];
}

@end
```

编译运行，你可以看到控制台会输出 `Logging` 
这里有几个地方需要特别留意下

1. 如果是交换的系统方法，在新的方法内部一定要再调用这个方法一次，因为这个时候方法的imp指针已经交换，调用该方法就是调用系统方法，为什么要这么做呢，因为这些系统的方法是黑盒的，有很多我们不清楚的操作，如果不调用可能会给程序带来问题
2. 执行交换方法的最佳时机是在类方法`load`中, 该方法会在类被加载的时候执行
3. 一定要使用GCD或其他办法保证交换的线程安全，仅执行一次，防止出现错误。

#### 关联对象

比如我们想要为 UIViewController 添加一个flag属性记录状态，但是无法更改 UIViewController，那么我们可以在 category 中添加属性


```objectivec

#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface UIViewController (Log)

@property (nonatomic ,copy) NSString *flag;

@end

NS_ASSUME_NONNULL_END

```

然后在其他的 viewController 中使用


```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.flag = @"active";
}
```

运行后可以看到崩溃 `unrecognized selector sent to instance` ，
这是因为在 category 中 property修饰符并不会自动为我们生成成员变量，而我们知道，属性其实是 ivar + getter & setter ，所以我们可以使用 runtime 来手动关联:

在 category 的 .m 文件中增加以下代码

```objectivec
- (void)setFlag:(NSString *)flag {
    objc_setAssociatedObject(self, @selector(flag), flag, OBJC_ASSOCIATION_COPY);
}

- (NSString *)flag {
    return objc_getAssociatedObject(self, _cmd);
}
```

然后就可以在其他 viewController 中随意使用了，由于 objc_setAssociatedObject 也是在ARC管理之下的所以我们也不必手动释放。

#### 写在最后

虽然 Runtime 有诸多魔幻的使用方法，但是不建议过多的使用(除非掌握的很熟练)，除非是开发框架，否则多个互相交换的方法和动态的属性在调试的时候会很无奈