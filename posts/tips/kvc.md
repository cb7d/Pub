# Key-value coding

> Key-value coding is a mechanism enabled by the NSKeyValueCoding informal protocol that objects adopt to provide indirect access to their properties. When an object is key-value coding compliant, its properties are addressable via string parameters through a concise, uniform messaging interface. This indirect access mechanism supplements the direct access afforded by instance variables and their associated accessor methods.

以上是Apple对KVC的定义，翻译过来就是：

> 键值编码是由NSKeyValueCoding非正式协议启用的机制，对象采用该机制提供对其属性的间接访问。 当对象符合键值编码时，其属性可通过字符串参数通过简洁，统一的消息传递接口寻址。 这种间接访问机制补充了实例变量及其相关访问器方法提供的直接访问。

所有直接或者间接继承NSObject的对象都遵守 **NSKeyValueCoding** 协议，并默认提供实现，找到 **NSKeyValueCoding.h** 我们可以看到里面为这些类添加了一些extension

```objectivec

@interface NSObject(NSKeyValueCoding)

@interface NSArray<ObjectType>(NSKeyValueCoding)

@interface NSDictionary<KeyType, ObjectType>(NSKeyValueCoding)

@interface NSMutableDictionary<KeyType, ObjectType>(NSKeyValueCoding)

@interface NSOrderedSet<ObjectType>(NSKeyValueCoding)

@interface NSSet<ObjectType>(NSKeyValueCoding)

```

下面是一些主要的方法：

```objectivec

- (nullable id)valueForKey:(NSString *)key;

- (void)setValue:(nullable id)value forKey:(NSString *)key;

- (nullable id)valueForKeyPath:(NSString *)keyPath;
  
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;

- (nullable id)valueForUndefinedKey:(NSString *)key;

- (void)setValue:(nullable id)value forUndefinedKey:(NSString *)key;

```

实践是检验真理的唯一标准，下面创建一个工程实践一下

1. Xcode -> New -> MacOS -> CommandLine
2. 创建Person类

关于KVC要分为两部分，集合类型与非集合类型，下面先来看**非集合类型**

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

打开main.m，创建两个Person的实例Alice和Bob

```objectivec
#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Person *Alice = Person.new;
        Alice.name = @"Alice";
        Alice.age = 18;
        Alice.friends = @[];
        
        Person *Bob = Person.new;
        [Bob setValue:@"Bob" forKey:@"name"];
        [Bob setValue:@(28) forKey:@"age"];
        [Bob setValue:@[Alice] forKey:@"friends"];
        
        NSLog(@"%@",@[Alice,Bob]);
        NSLog(@"%@",[Bob valueForKeyPath:@"friends"]);
    }
    return 0;
}

```

其中，ALice 使用正常的 set 方法进行赋值，Bob使用`KVC`方式赋值，
`Command` + R，能够看到如下输出

```objectivec
(
    "- name: Alice, age: 18, friends: (\n)",
    "- name: Bob, age: 28, friends: (\n    \"- name: Alice, age: 18, friends: (\\n)\"\n)"
)

(
    "- name: Alice, age: 18, friends: (\n)"
)
```

输出正如我们想的那样，通过KVC，正确的对Bob的name，age，friends进行了赋值和取值操作

那么若果想直接获取Bob名字的长度呢

```objectivec
id nameLength = [Bob valueForKeyPath:@"name.length"];
```

有没有发现这种方式很便捷呢，后面还有更有趣的用法。

接下来想一个问题，如果我想使用KVC进行赋值但是相应的类并没有对应的属性怎么办

```objectivec
[Bob setValue:@"Bob" forKey:@"familyName"];
```

运行就会崩溃掉，而在生产环境的情况下我们可不想让App Crash，
可以看下报错信息

```objectivec
 Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<Person 0x1030063b0> setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key familyName.'
```

可以发现导致崩溃的原因是Person类没有实现这个方法 setValue:forUndefinedKey:
接下来在Person.m里面添加这个方法的实现

```objectivec
- (void)setValue:(id)value forUndefinedKey:(NSString *)key {
    
}
```

可以看到运行之后不会崩溃了

上面简单介绍了一下非集合类型，下面先来看**集合类型**

首先new一个Array

```objectivec
NSArray *family = @[Alice,Bob];
NSLog(@"%@",[family valueForKey:@"count"]);
```

运行后发现又又又crash了，

```objectivec
*** Terminating app due to uncaught exception 'NSUnknownKeyException', reason: '[<Person 0x100703db0> valueForUndefinedKey:]: this class is not key value coding-compliant for the key count.'
```

按照正常的理解，我们无非是想回去Array的count属性，为什么会崩溃呢，原来对集合类型进行KVC操作时是分别对集合内对象进行操作，

```objectivec
NSLog(@"%@",[family valueForKey:@"name"]);
```

现在的输出是一个Array

```objectivec
(
    Alice,
    Bob
)
```

使用KVC对集合类型内部对象赋值也是一样的

```objectivec
NSArray *family = @[Alice,Bob];
[family setValue:@"NoName" forKey:@"name"];
NSLog(@"%@",family);
```

当然，你也可以自己重写上述的方法来使用，这里就不多赘述。

下面来还有一些更有趣的

### KVC集合操作符

![keypath](https://raw.githubusercontent.com/FelixScat/Pub/master/image/keypath.jpg)

可以看到对于集合操作，在keypath中间我们可以加入一个集合操作符，能够实现更为便捷的骚操作

集合操作符有一下几种

- @avg   // 算数平均值
- @count // 计数
- @max   // 最大值
- @min   // 最小值
- @sum   // 求和
- @unionOfObjects // 创建并返回一个数组，该数组包含与右键路径指定的属性对应的集合的所有对象。
- @distinctUnionOfObjects // 同unionOfObjects，但会对数组去重
- @unionOfArrays // 创建并返回一个数组，该数组包含与右键路径指定的属性对应的所有集合的组合的对象。
- @distinctUnionOfArrays // 同unionOfArrays，但会对数组去重

```objectivec
NSLog(@"%@",[family valueForKeyPath:@"@max.age"]);
NSLog(@"%@",[family valueForKeyPath:@"@min.age"]);
NSLog(@"%@",[family valueForKeyPath:@"@sum.age"]);
NSLog(@"%@",[family valueForKeyPath:@"@avg.age"]);
```

这里可以看到获取到了Alice和Bob年龄的最大值，最小值，和，平均值

还有一种特殊场景，就是我想获取这个family的所有friends怎么做呢

```objectivec
NSLog(@"%@",[family valueForKeyPath:@"@unionOfObjects.friends"]);

(
        (
    ),
        (
        "- name: Alice, age: 18, friends: (\n)"
    )
)
```

可以看到打印似乎有些不对劲，我们想要的是所有的friends，但数组中却是Alice的friends，Bob的friends两个对象。
这时候我们可以使用unionOfArrays，这个操作符会帮我们将数组元素【铺平】

```objectivec
NSLog(@"%@",[family valueForKeyPath:@"@unionOfArrays.friends"]);
```

一句代码即展开获取了你所需要的Array，否则你只能写一些胶水代码

```objectivec
NSArray *arr = [Alice.friends arrayByAddingObjectsFromArray:Bob.friends];
```

不够优雅

基本的KVC就介绍这么多，如果有更深层次的自定义实现需求的话还是推荐多看看Apple的文档，不过记得使用KVC的话当对象更新了property不要忘记更新keypath