# Timer 的一个小问题

> 开发过程中我们必不可少的需要接触定时器，在iOS中，常用的定时器有以下几种：

- GCD Timer
- CADisplayLink
- NSTimer

这里我们主要来看下 `NSTimer` 的一个问题

```objectivec
#import "ViewController.h"

@interface ViewController ()

@property (nonatomic, strong) NSTimer *t;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
}

- (void)startTImer {
    
    _t = [NSTimer timerWithTimeInterval:1.0f target:self selector:@selector(someBussiness) userInfo:nil repeats:true];
    
    [[NSRunLoop currentRunLoop] addTimer:_t forMode:NSRunLoopCommonModes];
}

- (void)someBussiness {
    
    NSLog(@"timer triggered");
}

- (void)dealloc {
    
    NSLog(@"Controller dealloc");
    
    if (self.t) {
        
        [self.t invalidate];
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (self.presentingViewController) {
        [self dismissViewControllerAnimated:true completion:nil];
    }else {
        
        ViewController *vc = ViewController.new;
        vc.view.backgroundColor = UIColor.grayColor;
        [self presentViewController:vc animated:true completion:nil];
        [vc startTImer];
    }
}

@end
```

这里在我们点击页面之后，会present出来一个新页面，并开始使用 NSTimer 计时，并在 dealloc 中打印信息。
再次点击present出来的viewController把当前的Controller销毁掉。

再次点击我们会发现，计时器并没有停止，而且预期的dealloc中的信息也并没有打印，这是为什么呢？

这里我们可以使用Xcode的 Debug Memory Graph ,就在下方控制台上面的按键里面，可以看到如图所示

![retainCircle](https://github.com/FelixScat/Pub/blob/master/image/retainCircle.png?raw=true)

我们可以看到这里 Runloop 引用了 timer ，而 timer 又引用了当前的Controller，最终导致Controller无法释放

我们通常会想，那把 NSTimer 的 property 用weak来修饰，或者把timer的target使用 weak 修饰不就好了吗。那我们来修改一下代码

```objectivec
@property (nonatomic, weak) NSTimer *t;

- (void)startTImer {
    
    __weak typeof(self) ws = self;
    
    _t = [NSTimer timerWithTimeInterval:1.0f target:ws selector:@selector(someBussiness) userInfo:nil repeats:true];
    
    [[NSRunLoop currentRunLoop] addTimer:_t forMode:NSRunLoopCommonModes];
}
```

这里我们修改timer的property为weak，把target也修饰为weak，再次运行。

哈，还是没有释放，timer 仍在打印。

这里其实是因为Runloop会对加入的Timer自动强引用， 而timer会对target进行强引用，即使修饰为weak也没用，那么，有咩有什么办法来释放他呢？

```objectivec
- (void)startTImer {
    
    __weak typeof(self) ws = self;
    
    _t = [NSTimer timerWithTimeInterval:1.0f repeats:true block:^(NSTimer * _Nonnull timer) {
        
        [ws someBussiness];
    }];
    
    [[NSRunLoop currentRunLoop] addTimer:_t forMode:NSRunLoopCommonModes];
}
```

😂 😂 😂 改为Block调用的方式就可以了，那么有没有别的方式也可以解决这个问题呢？（当然有了要不这篇我tm是在写啥）

### NSProxy

> An abstract superclass defining an API for objects that act as stand-ins for other objects or for objects that don’t exist yet.


> 一个抽象超类，用于定义对象的API，这些对象充当其他对象或尚不存在的对象的替身。

[官方文档](https://developer.apple.com/documentation/foundation/nsproxy)


使用NSProxy我们可以把任意的对象隐藏在后面，由这个抽象类在前面为我们真实的对象代理，当然，我们需要实现两个方法

```objectivec
- (void)forwardInvocation:(NSInvocation *)invocation;

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel;
```

第一个是方法决议，我们可以在这里改变方法的指针，更换方法，
第二个是方法签名，用来提供相应的函数返回类型和参数，

接下来我们新建 TimerProxy 类 继承 NSProxy

TimerProxy.h

```objectivec
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface TimerProxy : NSProxy

+ (instancetype)proxyWithObject:(id)obj;

@end

NS_ASSUME_NONNULL_END
```

TimerProxy.m

```objectivec
#import "TimerProxy.h"

@interface TimerProxy ()

@property (nonatomic, weak) id object;

@end

@implementation TimerProxy


- (instancetype)withProxy:(id)obj {
    
    _object = obj;
    
    return self;
}

+ (instancetype)proxyWithObject:(id)obj {
    
    return [[self alloc] withProxy:obj];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    
    SEL selector = invocation.selector;
    
    if ([_object respondsToSelector:selector]) {
        
        [invocation invokeWithTarget:_object];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [_object methodSignatureForSelector:sel];
}

@end
```

再更新一下viewController的实现

```objectivec
#import "ViewController.h"
#import "TimerProxy.h"

@interface ViewController ()

@property (nonatomic, strong) NSTimer *t;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
}

- (void)startTImer {
    
    _t = [NSTimer timerWithTimeInterval:1.0f target:[TimerProxy proxyWithObject:self] selector:@selector(someBussiness) userInfo:nil repeats:true];
    
    [[NSRunLoop currentRunLoop] addTimer:_t forMode:NSRunLoopCommonModes];
}

- (void)someBussiness {
    
    NSLog(@"timer triggered");
}

- (void)dealloc {
    
    NSLog(@"Controller dealloc");
    
    if (self.t) {
        
        [self.t invalidate];
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (self.presentingViewController) {
        [self dismissViewControllerAnimated:true completion:nil];
    }else {
        
        ViewController *vc = ViewController.new;
        vc.view.backgroundColor = UIColor.grayColor;
        [self presentViewController:vc animated:true completion:nil];
        [vc startTImer];
    }
}

@end
```

应该可以看到正常的dealloc的输出，并且timer也停止了，
NSProxy是一个非常有用的抽象类，当然还有其他用途，比如能够模拟多继承，后续会补充相关的整理资料。