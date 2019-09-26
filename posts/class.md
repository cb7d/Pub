# Class

```objectivec
@interface NSObject <NSObject> {
Class isa  OBJC_ISA_AVAILABILITY;
@end
```

ObjC是一门面向对象的语言，参考了Smalltalk的消息传递机制，那么对于开发者来说，对象、类、元类这些究竟是什么呢，他们之间的关系又是怎样的呢，这篇主要写一点对类结构的总结，偏向原理

## 实例对象

```objectivec
#import <Foundation/Foundation.h>

@interface Person: NSObject

@property (nonatomic, copy) NSString *name;

@end

@implementation Person

- (void)greeting {
    NSLog(@"Hi, I'm %@", self.name);
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        
        Person *person = [[Person alloc] init];
        person.name = @"Alice";
        
        [person greeting];
        
    }
    return 0;
}
```

我们首先实例化一个Person对象，然后调用setter为name赋值，再调用person的实例方法greeting

我们看一下使用clang转换之后的代码

```cpp
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 


        Person *person = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc")), sel_registerName("init"));
        ((void (*)(id, SEL, NSString *))(void *)objc_msgSend)((id)person, sel_registerName("setName:"), (NSString *)&__NSConstantStringImpl__var_folders_60_yqdp8_yn6vbd1tl6s3k479sr0000gp_T_main_5be332_mi_1);

        ((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("greeting"));

    }
    return 0;
}
```

代码的大体逻辑是这样的：

1. 使用objc_getClass获取Person的类对象
2. 向该类对象发送消息 alloc，得到返回值，也就是person对象
3. 发送init消息，初始化对象
4. 向对象发送 setName: 消息，将实例化的字符串作为参数传入
5. 向对象发送greeting消息

这比刚才写下的代码复杂了一些，这里不对消息发送做过多介绍，有兴趣的同学看我的另一篇总结

现在就需要探究一下子为啥person能够响应我们的方法了，掏出源码瞅瞅

(代码有删减)

```cpp
Class objc_getClass(const char *aClassName)
{
    if (!aClassName) return Nil;
    return look_up_class(aClassName, NO, YES);
}
```

```cpp
static Class getClass(const char *name)
{
    Class result = getClass_impl(name);
    if (result) return result;
    return nil;
}
```

我们发现这个方法最终会返回Class类型的对象，看起来，这就是对象能够响应方法的秘密了

## 类对象

进入Class的定义，我们能够看到以下代码

```cpp
typedef struct objc_class *Class;

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
  	bool isMetaClass() {
        assert(this);
        assert(isRealized());
        return data()->ro->flags & RO_META;
    }

    // NOT identical to this->ISA when this is a metaclass
    Class getMeta() {
        if (isMetaClass()) return (Class)this;
        else return this->ISA();
    }
    bool isRootClass() {
        return superclass == nil;
    }
    bool isRootMetaclass() {
        return ISA() == (Class)this;
    }
}
```

我们发现，类居然也是对象。每个对象都有一个指针`Class isa`指向自己所属的类，这个isa就是找到对象能响应的方法列表的关键

```cpp
- (IMP)methodForSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return object_getMethodImplementation(self, sel);
}

IMP object_getMethodImplementation(id obj, SEL name)
{
    Class cls = (obj ? obj->getIsa() : nil);
    return class_getMethodImplementation(cls, name);
}

IMP class_getMethodImplementation(Class cls, SEL sel)
{
    IMP imp;

    if (!cls  ||  !sel) return nil;

    imp = lookUpImpOrNil(cls, sel, nil, 
                         YES/*initialize*/, YES/*cache*/, YES/*resolver*/);

    // Translate forwarding function to C-callable external version
    if (!imp) {
        return _objc_msgForward;
    }

    return imp;
}
```

这里应该可以清楚上面的代码调用过程了

1. 创建对象的时候根据类名获取 isa，也就是类对象
2. 在向对象发送消息的时候使用 getIsa 方法获取类对象，在类对象中查找方法的实现，

这里就不对方法缓存做展开了

## 元类

到了这里还并没有真正的了解到类结构的底部，回到最开始的初始化person的地方，我们向Person的类对象发送了alloc消息，那么这条alloc消息是如何响应的呢？

```objectivec
struct objc_class : objc_object
```

objc_class也是有isa的，类对象的isa是用来描述类的，这就是metaclass元类了

- 类是对实例对象的描述
- 元类是对类对象的描述

所以，在向Person发送alloc消息的时候，实际上是在Person的类对象的元类中去查找方法实现，并发送消息的

- 实例方法要去类对象中去找
- 类方法要由元类去找

那么你可能会想，元类需不需要描述呢

我们可以看下NSObject.mm的实现

```objectivec
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}

+ (Class)superclass {
    return self->superclass;
}

- (Class)superclass {
    return [self class]->superclass;
}
```

可以看到，NSObject是一个根类，没有父类，当你向NSObject调用class方法，返回的指针还是指向NSObject的，所以可以总结为以下几点

- 根类的元类实际上是他自己
- 子类的元类使用根类的元类作为他的类（绕口令？）

从根类的元类开始，形成了一个环，到这里我们就对ObjC的对象、类结构清晰许多了

## 内省

把上面的这些连起来，再来谈谈内省

> 内省是对象揭示自己作为一个运行时对象的详细信息的一种能力。这些详细信息包括对象在继承树上的位置，对象是否遵循特定的协议，以及是否可以响应特定的消息。NSObject协议和类定义了很多内省方法，用于查询运行时信息，以便根据对象的特征进行识别。

下面是几个内省的方法

判断当前对象是否属于某个类或其父类

```cpp
- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

是否为此类的实例

```cpp
- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

对象是否能够响应某个方法

```cpp
- (BOOL)respondsToSelector:(SEL)sel {
    if (!sel) return NO;
    return class_respondsToSelector_inst([self class], sel, self);
}
```

对象是否遵守协议

```cpp
- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}
```

## 动态创建类

既然类也是对象，这就给了我们在运行时动态创建类的可能，看下怎么实现

```objectivec
void greeting(id self, SEL _cmd) {

    Ivar name = class_getInstanceVariable([self class], "name");
    NSLog(@"Hi, I'm %@", object_getIvar(self, name));
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        
        Class Person = objc_allocateClassPair([NSObject class], "Person", 0);
        
        BOOL success = class_addIvar(Person, "name", sizeof(NSString *), 0, @encode(NSString *));
        
        objc_registerClassPair(Person);
        
      	// TypeEncoding 
        class_addMethod(Person, NSSelectorFromString(@"greeting"), (IMP)greeting, "v@:");
        
        id person = [[Person alloc] init];
        
        [person setValue:@"Alice" forKey:@"name"];
        
        [person performSelector:NSSelectorFromString(@"greeting")];
        
    }
    return 0;
}

```

上面的TypeEncoding部分[可以参考](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

> Thanks





























