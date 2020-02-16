# Aspects

[本文示例工程](https://github.com/FelixScat/demo_Aspects)

什么是AOP？

**面向切面的程序设计**（Aspect-oriented programming，AOP，又译作**面向方面的程序设计**、**剖面导向程序设计**）是[计算机科学](https://zh.wikipedia.org/wiki/计算机科学)中的一种[程序设计思想](https://zh.wikipedia.org/wiki/编程范型)，旨在将**横切关注点**与业务主体进行进一步分离，以提高程序代码的模块化程度。通过在现有代码基础上增加额外的**通知**（Advice）机制，能够对被声明为“**切点**（Pointcut）”的代码块进行统一管理与装饰，如“对所有方法名以‘set*’开头的方法添加后台日志”。该思想使得开发人员能够将与代码核心业务逻辑关系不那么密切的功能（如日志功能）添加至程序中，同时又不降低业务代码的可读性。面向切面的程序设计思想也是面向切面软件开发的基础。

在开发过程中我们总会遇到某种需求，需要对我们业务内部的所有状态进行统一管理，比如对点击事件，用户进入的页面等进行埋点处理，对于这种需求我们一般会想到利用 **Runtime** 的消息转发功能实现这种需求，对这块不熟悉的同学可以[看这篇](https://k.felixplus.top/runtime/)，下面我们来看下 **Aspects** 是如何设计的

## 使用方式

先看下接口里面的方法

```objectivec
@interface NSObject (Aspects)

/// Adds a block of code before/instead/after the current `selector` for a specific class.
///
/// @param block Aspects replicates the type signature of the method being hooked.
/// The first parameter will be `id<AspectInfo>`, followed by all parameters of the method.
/// These parameters are optional and will be filled to match the block signature.
/// You can even use an empty block, or one that simple gets `id<AspectInfo>`.
///
/// @note Hooking static methods is not supported.
/// @return A token which allows to later deregister the aspect.
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

/// Adds a block of code before/instead/after the current `selector` for a specific instance.
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

@end
```

注意方法中的注释中特意提到了目前不支持静态方法的hook，方便的一点是 Aspect 需要传入的block是id类型，我们定义block的时候可以声明传递所有需要的参数，也可以什么都不传。

基本的使用方式

```objectivec

@implementation NSObject (Track)

+ (void)load {
    
    [UIViewController aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
        NSLog(@"View Controller %@ will appear animated: %tu", aspectInfo.instance, animated);
    } error:NULL];
}

@end
```

上面是返回类型为 **void** 的方法，需要返回参数的hook方式稍微复杂一点

```objectivec
#import "FKViewController.h"
#import <Aspects/Aspects.h>

@interface FKViewController ()

@end

@implementation FKViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
	// Do any additional setup after loading the view, typically from a nib.
    
    [self aspect_hookSelector:@selector(giveMeFive) withOptions:AspectPositionInstead usingBlock:^(id<AspectInfo> info) {
        // Call original implementation.
        NSNumber *number;
        NSInvocation *invocation = info.originalInvocation;
        [invocation invoke];
        [invocation getReturnValue:&number];
        
        if (number) {
            number = @(10);
            [invocation setReturnValue:&number];
        }
        
    } error:NULL];
}

- (NSNumber *)giveMeFive {
    return @(5);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    NSLog(@"%@", [self giveMeFive]);
}

@end
```



## 深入源码

### Aspects 定义的结构

#### Hook选项

```objectivec
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// 在原方法后调用(默认)
    AspectPositionInstead = 1,            /// 直接替换
    AspectPositionBefore  = 2,            /// 在原方法之前
    
    AspectOptionAutomaticRemoval = 1 << 3 /// 第一次执行后取消
};
```

这里并没有使用枚举，而是使用了 NS_OPTIONS，方便组合选项

#### 提供销毁Hook的实例

```objectivec
@protocol AspectToken <NSObject>

/// Deregisters an aspect.
/// @return YES if deregistration is successful, otherwise NO.
- (BOOL)remove;

@end
```

用协议的形式向外部暴露一个可销毁的选项，我们看下如何销毁执行的hook

```objectivec
@interface FKViewController ()

@property (nonatomic, strong) id<AspectToken> token;

@end

@implementation FKViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
	// Do any additional setup after loading the view, typically from a nib.
    
    __weak typeof(self) weakSelf = self;
    _token = [self aspect_hookSelector:@selector(giveMeFive) withOptions:AspectPositionInstead usingBlock:^(id<AspectInfo> info) {
        // Call original implementation.
        NSNumber *number;
        NSInvocation *invocation = info.originalInvocation;
        [invocation invoke];
        [invocation getReturnValue:&number];
        
        if (number) {
            number = @(10);
            [invocation setReturnValue:&number];
            [weakSelf.token remove];
        }
        
    } error:NULL];
}

- (NSNumber *)giveMeFive {
    return @(5);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    NSLog(@"%@", [self giveMeFive]);
}

@end
```

运行后多次点击屏幕，只有第一次会输出10了

#### 定义错误类型

```objectivec
typedef NS_ENUM(NSUInteger, AspectErrorCode) {
    AspectErrorSelectorBlacklisted,                   /// 无法hook的方法，release, retain等
    AspectErrorDoesNotRespondToSelector,              /// 找不到方法
    AspectErrorSelectorDeallocPosition,               /// hook dealloc方法只能选择在执行该方法前
    AspectErrorSelectorAlreadyHookedInClassHierarchy, /// 在子类中hook的方法已经被hook过
    AspectErrorFailedToAllocateClassPair,             /// objc_allocateClassPair 失败
    AspectErrorMissingBlockSignature,                 /// hook回调的block没有签名
    AspectErrorIncompatibleBlockSignature,            /// 签名不匹配

    AspectErrorRemoveObjectAlreadyDeallocated = 100   /// hook已经被移除
};
```

#### 定义返回block

```objectivec
typedef struct _AspectBlock {
	__unused Class isa;
	AspectBlockFlags flags;
	__unused int reserved;
	void (__unused *invoke)(struct _AspectBlock *block, ...);
	struct {
		unsigned long int reserved;
		unsigned long int size;
		// requires AspectBlockFlagsHasCopyDisposeHelpers
		void (*copy)(void *dst, const void *src);
		void (*dispose)(const void *);
		// requires AspectBlockFlagsHasSignature
		const char *signature;
		const char *layout;
	} *descriptor;
	// imported variables
} *AspectBlockRef;
```

#### 定义返回的Aspect信息

```objectivec
@protocol AspectInfo <NSObject>

/// The instance that is currently hooked.
- (id)instance;

/// The original invocation of the hooked method.
- (NSInvocation *)originalInvocation;

/// All method arguments, boxed. This is lazily evaluated.
- (NSArray *)arguments;

@end
```

#### 定义用来追踪单个 aspect 的 **AspectIdentifier**

```objectivec
// Tracks a single aspect.
@interface AspectIdentifier : NSObject
+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error;
- (BOOL)invokeWithInfo:(id<AspectInfo>)info;
@property (nonatomic, assign) SEL selector;
@property (nonatomic, strong) id block;
@property (nonatomic, strong) NSMethodSignature *blockSignature;
@property (nonatomic, weak) id object;
@property (nonatomic, assign) AspectOptions options;
@end
```

#### 负责追踪类的类

```objectivec
@interface AspectTracker : NSObject
- (id)initWithTrackedClass:(Class)trackedClass parent:(AspectTracker *)parent;
@property (nonatomic, strong) Class trackedClass;
@property (nonatomic, strong) NSMutableSet *selectorNames;
@property (nonatomic, weak) AspectTracker *parentEntry;
@end
```

#### aspect 的容器

```objectivec
// Tracks all aspects for an object/class.
@interface AspectsContainer : NSObject
- (void)addAspect:(AspectIdentifier *)aspect withOptions:(AspectOptions)injectPosition;
- (BOOL)removeAspect:(id)aspect;
- (BOOL)hasAspects;
@property (atomic, copy) NSArray *beforeAspects;
@property (atomic, copy) NSArray *insteadAspects;
@property (atomic, copy) NSArray *afterAspects;
@end
```

------

### 具体实现

下面我们一步一步，了解一下 Aspects 完整的运作过程

```objectivec
/// @return A token which allows to later deregister the aspect.
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add(self, selector, options, block, error);
}
```

类方法也是类似的原理，这里进入了 **aspect_add** 这个方法

```objectivec
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
  	// 首先，先对参数内容进行检查，不要小瞧这一步，没个方法调用都对自己的参数进行检查才不容易出错
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;
  	// 使用 OSSpinLock自旋锁 加锁 ，虽然 OSSpinLock 效率高一些
    aspect_performLocked(^{
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
          	// 获取aspect容器
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
          	// 生成追踪的对象
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception.
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```

这里要注意，虽然自旋锁的效率高一些，但是也有潜在的风险，详情看 [不再安全的OSSpinLock](http://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

容器使用runtime动态绑定在对象上

```objectivec
static AspectsContainer *aspect_getContainerForObject(NSObject *self, SEL selector) {
    NSCParameterAssert(self);
    SEL aliasSelector = aspect_aliasForSelector(selector);
    AspectsContainer *aspectContainer = objc_getAssociatedObject(self, aliasSelector);
    if (!aspectContainer) {
        aspectContainer = [AspectsContainer new];
        objc_setAssociatedObject(self, aliasSelector, aspectContainer, OBJC_ASSOCIATION_RETAIN);
    }
    return aspectContainer;
}
```

下面开始真正的hook部分

```objectivec
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
  	// 获取hook的class，所有的操作都在子类上进行，这样方便我们销毁hook时恢复isa，不造成影响
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
  	// 判断当前的imp是否为消息转发，如果不是消息转发就进行编码，添加方法并进行方法交换
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```

Hook 的部分告一段落，最后还有销毁 Hook 的收尾工作

```objectivec
static BOOL aspect_remove(AspectIdentifier *aspect, NSError **error) {
  	// assert AspectIdentifier
    NSCAssert([aspect isKindOfClass:AspectIdentifier.class], @"Must have correct type.");

    __block BOOL success = NO;
    aspect_performLocked(^{
        id self = aspect.object; // strongify
        if (self) {
          	// 取出容器，从容器中删除 aspect
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, aspect.selector);
            success = [aspectContainer removeAspect:aspect];
						
            aspect_cleanupHookedClassAndSelector(self, aspect.selector);
            // destroy token
            aspect.object = nil;
            aspect.block = nil;
            aspect.selector = NULL;
        }else {
          	// 当前对象已经释放
            NSString *errrorDesc = [NSString stringWithFormat:@"Unable to deregister hook. Object already deallocated: %@", aspect];
            AspectError(AspectErrorRemoveObjectAlreadyDeallocated, errrorDesc);
        }
    });
    return success;
}
```

对使用runtime修改的东西进行还原

```objective-c
// Will undo the runtime changes made.
static void aspect_cleanupHookedClassAndSelector(NSObject *self, SEL selector) {
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
	
	Class klass = object_getClass(self);
    BOOL isMetaClass = class_isMetaClass(klass);
    if (isMetaClass) {
        klass = (Class)self;
    }
		
    // 方法交换还原
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Restore the original method implementation.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        Method originalMethod = class_getInstanceMethod(klass, aliasSelector);
        IMP originalIMP = method_getImplementation(originalMethod);
        NSCAssert(originalMethod, @"Original implementation for %@ not found %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);

        class_replaceMethod(klass, selector, originalIMP, typeEncoding);
        AspectLog(@"Aspects: Removed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }

    // Deregister global tracked selector
    aspect_deregisterTrackedSelector(self, selector);

    // 检查并清理container
    AspectsContainer *container = aspect_getContainerForObject(self, selector);
    if (!container.hasAspects) {
        // Destroy the container
        aspect_destroyContainerForObject(self, selector);

        // Figure out how the class was modified to undo the changes.
        NSString *className = NSStringFromClass(klass);
        if ([className hasSuffix:AspectsSubclassSuffix]) {
            Class originalClass = NSClassFromString([className stringByReplacingOccurrencesOfString:AspectsSubclassSuffix withString:@""]);
            NSCAssert(originalClass != nil, @"Original class must exist");
            object_setClass(self, originalClass);
            AspectLog(@"Aspects: %@ has been restored.", NSStringFromClass(originalClass));

            // We can only dispose the class pair if we can ensure that no instances exist using our subclass.
            // Since we don't globally track this, we can't ensure this - but there's also not much overhead in keeping it around.
            //objc_disposeClassPair(object.class);
        }else {
            // Class is most likely swizzled in place. Undo that.
            if (isMetaClass) {
                aspect_undoSwizzleClassInPlace((Class)self);
            }
        }
    }
}
```

## 结语

这一套下来比我们随手写的要复杂一些，但是 Aspects 也并不是完美的，同样存在一些坑点，比如当已经使用了其他方法交换的时候再次对该方法使用 Aspects ，所以使用的时候一定要注意，这里只是梳理和总结下代码的结构和逻辑，回头打算补充一张流程图😄