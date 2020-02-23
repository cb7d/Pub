---
title: "浅谈 ObjC 中的几种copy"
date: 2018-06-19T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC","memory"]
---

# Copy

拷贝是我们在开发中经常使用的技巧，这里指的不是到Github上去复制粘贴代码，而是对内存中对象的操作 (逃

> 深拷贝与浅拷贝的区别 ？ 深拷贝是指我们拷贝出来的对象拥有自己单独的内存地址，修改新对象不影响源对象，浅拷贝指的是在copy指针的引用，修改新对象会影响到源对象

在ObjC里面主要有两个方法对对象进行拷贝

```objectivec
- (id)copy;
- (id)mutableCopy;
```

要对象能够使用这两个方法需要遵守协议 NSCopying, NSMutableCopying

那么，该何时使用这两种方法呢， 
先说结论，只有不可变对象调用copy方法的时候才是浅拷贝，其他情况均为深拷贝
新建一个工程验证一下吧

Xcode -> New -> MacOS -> CommandLine -> main.m

由于NSString 同时实现了 NSCopying, NSMutableCopying 两个协议，我们就用他来做实验

```objectivec
NSString *str1 = @"str1";
NSString *str2 = str1.copy;
NSString *str3 = str1.mutableCopy;

NSLog(@"%p %p %p",str1, str2, str3);
```

运行之后可以看到如下输出

```objectivec
0x1000020b8 0x1000020b8 0x100508e00
```

由此可以得出结论，不可变对象使用 mutableCopy 为深拷贝 ，copy 为浅拷贝

下面验证一下可变对象 NSMutableString

```objectivec
NSMutableString *str1 = [NSMutableString stringWithFormat:@"str1"];
NSString *str2 = str1.copy;
NSString *str3 = str1.mutableCopy;

NSLog(@"%p %p %p",str1, str2, str3);

0x100505260 0x50602036b4ec25f 0x100505290
```

得出结论，可变对象无论使用 copy 还是 mutableCopy ，均为深拷贝

这些是非集合类型的拷贝结论，那么对于集合类型来说呢

新建Person类

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

// 重写以便打印对象的属性
- (NSString *)description {
    
    return [NSString stringWithFormat:@"- name: %@, age: %ld, friends: %@",self.name, self.age, self.friends];
}

@end
```

如果正常按照上面的来想，集合类型应该也是一样 只有不可变对象调用copy方法的时候才是浅拷贝，其他情况均为深拷贝

下面创建Array来验证想法

```objectivec
Person *Alice = Person.new;
Alice.name = @"Alice";
Alice.age = 18;

NSArray *arr1 = @[Alice];
NSArray *arr2 = arr1.copy;
NSArray *arr3 = arr1.mutableCopy;

NSLog(@"%p %p %p",arr1, arr2, arr3);

0x1030064c0 0x1030064c0 0x103004a90
```

NSArray吻合我们上面的结论，下面看下NSMutableArray

```objectivec
Person *Alice = Person.new;
Alice.name = @"Alice";
Alice.age = 18;

NSMutableArray *arr1 = [NSMutableArray arrayWithArray:@[Alice]];
NSArray *arr2 = arr1.copy;
NSArray *arr3 = arr1.mutableCopy;

NSLog(@"%p %p %p",arr1, arr2, arr3);

0x1030051f0 0x103006a00 0x1030054a0
```

也温和上面的结论，但是稍等

```objectivec
NSMutableArray *arr1 = [NSMutableArray arrayWithArray:@[Alice]];
NSArray *arr2 = arr1.copy;

NSLog(@"%p %p",arr1.firstObject, arr2.firstObject);
```

我们可以打印arr1 和 arr2 的第一个元素的地址，发现他们还是指向同一个地址！

查看Apple关于[DeepCopy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Articles/Copying.html#//apple_ref/doc/uid/TP40010162-SW3 "DeepCopy")的文档

我们发现，原来要对一个集合类型实现真正的深拷贝需要用这种方法, 而上面这种官方名称为【单层深拷贝】

```objectivec
NSArray* trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData:
[NSKeyedArchiver archivedDataWithRootObject:oldArray]];
```

好，那么我们来改造代码

```objectivec
Person *Alice = Person.new;
Alice.name = @"Alice";
Alice.age = 18;

NSError *err;

NSMutableArray *arr1 = [NSMutableArray arrayWithArray:@[Alice]];
NSArray *arr2 = [NSKeyedUnarchiver unarchivedObjectOfClass:[NSArray class] fromData:[NSKeyedArchiver archivedDataWithRootObject:arr1 requiringSecureCoding:false error:&err] error:&err];

if (err) {
    NSLog(@"%@",err);
}

NSLog(@"%p %p",arr1.firstObject, arr2.firstObject);
```

奇怪的是输出arr2为nil，看一下控制台输出

```objectivec
-[Person encodeWithCoder:]: unrecognized selector sent to instance 0x100706ac0
```

原来是我们的Person没有遵守协议 NSCoding , 除此之外我们还需要遵守 NSSecureCoding 协议

改造后的Person.m

```objectivec
#import "Person.h"

@interface Person () <NSCoding, NSSecureCoding>

@end

@implementation Person

+ (BOOL)supportsSecureCoding {
    return true;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    
    [aCoder encodeObject:_name forKey:@"name"];
    [aCoder encodeInteger:_age forKey:@"age"];
}

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [super init];
    if (self) {
        self.name = [aDecoder decodeObjectForKey:@"name"];
        self.age = [aDecoder decodeIntegerForKey:@"age"];
    }
    return self;
}

- (instancetype)init {
    
    self = [super init];
    
    if (self) {
        _name = @"";
        _age = 0;
        _friends = @[];
    }
    
    return self;
}

```

现在我们打开main.m试一下

```objectivec
Person *Alice = Person.new;
Alice.name = @"Alice";
Alice.age = 18;

NSError *err;

NSMutableArray *arr1 = [NSMutableArray arrayWithArray:@[Alice]];
NSArray *arr2 = [NSKeyedUnarchiver unarchivedObjectOfClasses:[NSSet setWithObjects:[NSArray class],[Person class], nil] fromData:[NSKeyedArchiver archivedDataWithRootObject:arr1 requiringSecureCoding:true error:&err] error:&err];

if (err) {
    NSLog(@"%@",err);
}

NSLog(@"%p %p",arr1.firstObject, arr2.firstObject);

0x100709880 0x100705c60
```

我们发现数组里面的对象地址也全都不一样了，这才是集合类型的真正深拷贝

thanks.