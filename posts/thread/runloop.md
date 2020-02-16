# RunLoop

什么是RunLoop？

> A RunLoop object processes input for sources such as mouse and keyboard events from the window system, Port objects, and NSConnection objects. A RunLoop object also processes Timer events.
> Your application neither creates or explicitly manages RunLoop objects. Each Thread object—including the application’s main thread—has an RunLoop object automatically created for it as needed. If you need to access the current thread’s run loop, you do so with the class method current.
> Note that from the perspective of RunLoop, Timer objects are not "input"—they are a special type, and one of the things that means is that they do not cause the run loop to return when they fire.

[官方文档](https://developer.apple.com/documentation/foundation/runloop)

翻译如下

> RunLoop对象处理来自窗口系统，Port对象和NSConnection对象的源（如鼠标和键盘事件）的输入。 RunLoop对象还处理Timer事件。 您的应用程序既不创建也不显式管理RunLoop对象。每个Thread对象（包括应用程序的主线程）都会根据需要自动为其创建RunLoop对象。如果需要访问当前线程的运行循环，可以使用类方法current。 请注意，从RunLoop的角度来看，Timer对象不是“输入” - 它们是一种特殊类型，其中一个意思是它们不会导致运行循环在它们触发时返回。


我们都知道我们的应用是一个进程，在应用程序周期中我们大多都会开辟不同的线程来处理一些耗时的事情，在这里面Runloop扮演的是什么角色呢？

Runloop 可以翻译为 `事件循环` 我们可以把其理解为一个 while 循环

```swift
do {
    // something
}while()
```

Runloop是和线程一一对应的，其中，主线程的Runloop会在应用启动的时候进行启动和绑定，其他线程的Runloop要通过手动调用 `[NSRunLoop currentRunLoop]` 才会生成

看一下Foundation中的NSRunLoop.h, 里面有这几个方法需要注意一下:


```objectivec

@property (class, readonly, strong) NSRunLoop *currentRunLoop;
@property (class, readonly, strong) NSRunLoop *mainRunLoop;
@property (nullable, readonly, copy) NSRunLoopMode currentMode;

- (void)addTimer:(NSTimer *)timer forMode:(NSRunLoopMode)mode;

- (void)addPort:(NSPort *)aPort forMode:(NSRunLoopMode)mode;

```

其中 currentRunLoop 表示获取当前的Runloop ，mainRunLoop代表获取主事件循环，这里可以看出，非主线程的Runloop必须在子线程内获取，而mainRunLoop可以在任意线程获取。

NSRunLoopMode 在官方文档中提到的有五个:

- NSDefaultRunLoopMode
- NSConnectionReplyMode
- NSModalPanelRunLoopMode
- NSEventTrackingRunLoopMode
- NSRunLoopCommonModes

其中公开暴露出来的只有 NSDefaultRunLoopMode 和 NSRunLoopCommonModes ，在这里面 NSRunLoopCommonModes 实际上包含了 NSDefaultRunLoopMode 和 NSEventTrackingRunLoopMode，在需要追踪包括scrollview滚动等事件的时候最好使用 NSRunLoopCommonModes


## Runloop 的实际应用

首先，我们可以在viewcontroller中的touchbegin方法打断点，看一下该方法的调用栈

```objectivec
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    
}
```

在断点出发时我们在控制台查看

```objectivec
(lldb) thread backtrace
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
  * frame #0: 0x000000010a1589a0 Library_test`-[ViewController touchesBegan:withEvent:](self=0x00007f7fda41adc0, _cmd="touchesBegan:withEvent:", touches=1 element, event=0x0000600002649440) at ViewController.m:35
    frame #1: 0x0000000110e088e8 UIKitCore`forwardTouchMethod + 353
    frame #2: 0x0000000110e08776 UIKitCore`-[UIResponder touchesBegan:withEvent:] + 49
    frame #3: 0x0000000110e17dff UIKitCore`-[UIWindow _sendTouchesForEvent:] + 2052
    frame #4: 0x0000000110e197a0 UIKitCore`-[UIWindow sendEvent:] + 4080
    frame #5: 0x0000000110df7394 UIKitCore`-[UIApplication sendEvent:] + 352
    frame #6: 0x0000000110ecc5a9 UIKitCore`__dispatchPreprocessedEventFromEventQueue + 3054
    frame #7: 0x0000000110ecf1cb UIKitCore`__handleEventQueueInternal + 5948
    frame #8: 0x000000010cb87721 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17
    frame #9: 0x000000010cb86f93 CoreFoundation`__CFRunLoopDoSources0 + 243
    frame #10: 0x000000010cb8163f CoreFoundation`__CFRunLoopRun + 1263
    frame #11: 0x000000010cb80e11 CoreFoundation`CFRunLoopRunSpecific + 625
    frame #12: 0x00000001143891dd GraphicsServices`GSEventRunModal + 62
    frame #13: 0x0000000110ddb81d UIKitCore`UIApplicationMain + 140
    frame #14: 0x000000010a1590d0 Library_test`main(argc=1, argv=0x00007ffee5aa7000) at main.m:14
    frame #15: 0x000000010e331575 libdyld.dylib`start + 1
(lldb) 
```

可以看到 `CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION` 的方法调用

其实这就是一个点击的事件触发了，通过runloop传递的例子，实际上所有的事件都会通过这个方法调用，最后触发到我们编写的代码, 理解了这个我们接下来看看具体的应用。

### NSTimer

在controller中添加一个timer

```objectivec
#import "ViewController.h"

@interface ViewController ()

@property (nonatomic, strong) NSTimer *t;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _t = [NSTimer timerWithTimeInterval:1.0f repeats:true block:^(NSTimer * _Nonnull timer) {

        NSLog(@"Timer Triggered");
    }];
    [[NSRunLoop currentRunLoop] addTimer:_t forMode:NSRunLoopCommonModes];
}

@end
```

### 处理耗时逻辑

在后台进程处理逻辑，完成后通过modes避免在滑动视图的时候返回

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self performSelectorInBackground:@selector(someBussiness) withObject:nil];
}

- (void)someBussiness {
    
    [self performSelector:@selector(bussinessFinished) withObject:nil afterDelay:0.0f inModes:@[NSDefaultRunLoopMode]];
}

- (void)bussinessFinished {
    
}
```

### 自定义后台线程

有时我们想把一些逻辑全部放到自定义的线程中去处理，下面给出一个解决方案。

```objectivec
#import "ViewController.h"

@interface ViewController ()

@property (nonatomic, strong) NSThread *thread;

@property (nonatomic, strong) NSPort *port;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.thread start];
}

- (void)selfThread {
    
    @autoreleasepool {
    
        [[NSRunLoop currentRunLoop] run];
    }
}

- (NSThread *)thread {
    if (!_thread) {
        _thread = [[NSThread alloc] initWithTarget:self selector:@selector(selfThread) object:nil];
      	[_thread start];
    }
    return _thread;
}

- (NSPort *)port {
    if (!_port) {
        _port = [NSPort port];
    }
    return _port;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    [self performSelector:@selector(someBussiness) onThread:self.thread withObject:nil waitUntilDone:false];
}

- (void)someBussiness {
    
}

@end
```

运行之后貌似点击屏幕并没有反应？

这是因为我们没有为Runloop加入source（输入源），source可以是timer，port。
其中，NSPort是一个描述通信通道的抽象类，我们可以先使用port作为输入源来让Runloop持续运行，改造这个方法

```objectivec
- (void)selfThread {
    
    @autoreleasepool {
        [[NSRunLoop currentRunLoop] addPort:self.port forMode:NSRunLoopCommonModes];
        [[NSRunLoop currentRunLoop] run];
    }
}
```

现在我们就可以把需要执行的业务逻辑放入自定义的线程中执行了

还有其他许多实际的应用，比如优化空闲状体的利用、集合视图的优化等等，这里就不多叙述了