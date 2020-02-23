---
title: "KVO 键值监听"
date: 2019-04-17T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC"]
---

# Key-Value Observing

键值观察，是设计模式中观察者模式的实现

> 键值观察提供了一种机制，允许对象通知其他对象的特定属性的更改。它对应用程序中模型和控制器层之间的通信特别有用。（在OS X中，控制器层绑定技术严重依赖于键值观察。）控制器对象通常观察模型对象的属性，视图对象通过控制器观察模型对象的属性。然而，另外，模型对象可以观察其他模型对象（通常用于确定从属值何时改变）或甚至自身（再次确定从属值何时改变）。

## 使用KVO

Xcode -> New -> MacOS -> CommandLine 新建工程，创建Person类

Person.h

```objectivec
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject

@property (nonatomic ,copy) NSString *name;

@property (nonatomic ,assign) NSUInteger age;

@property (nonatomic ,copy) NSArray<Person *> *friends;

@end

NS_ASSUME_NONNULL_END
```

Person.m

```objectivec
#import "Person.h"

@implementation Person

- (instancetype)init {
    
    self = [super init];
    
    if (self) {
        _name = @"";
        _age = 0;
        _friends = @[];
    }
    
    return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    
    // 当收到通知的时候打印观察的对象，旧值和新值
    NSLog(@"\nReceving ObserveValueChanged \nObject: %@ OldValue: %@, NewValue: %@",object, change[NSKeyValueChangeOldKey], change[NSKeyValueChangeNewKey]);
}

// 重写以便打印对象的属性
- (NSString *)description {
    
    return [NSString stringWithFormat:@"- name: %@, age: %ld, friends: %@",self.name, self.age, self.friends];
}

@end
```

打开main.m，创建Alice和Bob，设置Bob观察Alice的age属性

```objectivec
#import <Foundation/Foundation.h>
#import "Person.h"

static void *PersonChangeContext = &PersonChangeContext;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        
        Person *Alice = Person.new;
        Alice.name = @"Alice";
        Alice.age = 18;
        
        Person *Bob = Person.new;
        Bob.name = @"Bob";
        Bob.age = 28;
        
        
        [Alice addObserver:Bob forKeyPath:@"age" options:NSKeyValueObservingOptionOld|NSKeyValueObservingOptionNew context:PersonChangeContext];
        
        Alice.age = 100;
        
        [Alice removeObserver:Bob forKeyPath:@"age"];
        
        Alice.age = 200;
        
    }
    return 0;
}
```

可以看到控制台输出了一次Alice的age属性的前后变化.

其中，NSKeyValueObservingOptions 有以下几个选项，可以使用 | 符号组合使用

- NSKeyValueObservingOptionNew // 收到通知时change字典将包含新值
- NSKeyValueObservingOptionOld // 收到通知时change字典将包含旧值
- NSKeyValueObservingOptionInitial // 在addObserver时会发送通知，change字典将包含初始值
- NSKeyValueObservingOptionPrior // 在所观察keyPath改变之前将收到通知

change字典的key值在这里

```objectivec
FOUNDATION_EXPORT NSKeyValueChangeKey const NSKeyValueChangeKindKey;
FOUNDATION_EXPORT NSKeyValueChangeKey const NSKeyValueChangeNewKey;
FOUNDATION_EXPORT NSKeyValueChangeKey const NSKeyValueChangeOldKey;
FOUNDATION_EXPORT NSKeyValueChangeKey const NSKeyValueChangeIndexesKey;
FOUNDATION_EXPORT NSKeyValueChangeKey const NSKeyValueChangeNotificationIsPriorKey 
```

KVO的使用场景有很多，比如Person拥有一个account属性，person需要获取account的变化该如何处理呢，轮训或许是个办法，但是无疑效率低下，而且很不安全，更合理的办法就是使用KVO观察account，在account发生变化时更新。

## 原理

现在我们已经知其然了，但是还不知其所以然。

### 先说结论

1. 系统通过runtime生成继承与被观察者的新类(NSKVONotifying_Person)，对原对象进行isa-swizzing(isa混写)
2. 根据KVC(键值编码)对对象的keypath进行hock
3. 在将要对属性进行赋值操作时发送通知

如何证明上述结论呢，上代码！

```objectivec
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "Person.h"

static void *PersonChangeContext = &PersonChangeContext;

int main(int argc, const char * argv[]) {
    @autoreleasepool {        
        
        Person *Alice = Person.new;
        Alice.name = @"Alice";
        Alice.age = 18;
        
        Person *Bob = Person.new;
        Bob.name = @"Bob";
        Bob.age = 28;
        
        
        Class cls0 = object_getClass(Alice);
        
        [Alice addObserver:Bob forKeyPath:@"age" options:NSKeyValueObservingOptionOld|NSKeyValueObservingOptionNew context:PersonChangeContext];
        
        Alice.age = 100;
        
        Class cls1 = object_getClass(Alice);
        
        NSLog(@"%@ %@",cls0,cls1);
        
    }
    return 0;
}
```

1. 首先引入runtime，这样我们可以打印isa所指向的真实的类，
2. 在向Alice添加观察者之前，先获取Class
3. 在向Alice添加观察者之后，获取Class

```objectivec
Person NSKVONotifying_Person
```

可以看到输出确实如此

### 如何禁止对象被观察?

我们只要在Person类中重写这个方法，外部就无法对我们Person的属性进行监听啦

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    return false;
}
```

### 如何单独禁止某个属性被观察？

我们在Person里实现automaticallyNotifiesObserversOf + XXX（属性名），并返回为 false 就可以单独禁止自动为某个属性生成监听了

```objectivec
+ (BOOL)automaticallyNotifiesObserversOfAge {
    return false;
}
```

### 如何自定义KVO触发条件？

有的时候我们只是需要在属性改变为某个状态的时候才需要作出响应，这个时候我们可以在上面方法的基础上自定义发送通知

```objectivec
- (void)setAge:(NSUInteger)age {
    if (age > 33) {
        [self willChangeValueForKey:NSStringFromSelector(@selector(age))];
    }
    _age = age;
    if (age > 33) {
        [self didChangeValueForKey:NSStringFromSelector(@selector(age))];
    }
}
```

比如在age的setter里面，判断condition，如果达成条件则调用 `willChangeValueForKey` 和 `didChangeValueForKey` 通知外部。

这就是KVO的核心思路了，关于KVC请看[这里](https://k.felixplus.top/kvc/).