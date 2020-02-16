# NSCache

NSCache 基本上就是一个会自动移除对象来释放内存的 NSMutableDictionary。无需响应内存警告或者使用计时器来清除缓存。唯一的不同之处是键对象不会像 NSMutableDictionary 中那样被复制，这实际上是它的一个优点（键不需要实现 NSCopying 协议）。

### 先列一下使用NSCache的好处

- NSCache是一个类似NSDictionary一个可变的集合。
- 提供了可设置缓存的数目与内存大小限制的方式。
- 保证了处理的数据的线程安全性。
- 缓存使用的key不需要是实现NSCopying。
- 当内存警告时内部自动清理部分缓存数据。

### NSCache的属性与方法

```objectivec
@property (assign) id<NSCacheDelegate>delegate;
```

cache对象的代理 ， 用来即将清理cache的时候得到通知

```objectivec
- (void)cache:(NSCache *)cache willEvictObject:(id)obj;
```

代理方法 ， 这里面不要对cache进行改动 ， 如果对象obj需要被持久化存储的话可以在这里进行操作

这里面有几种情况会导致该方法执行：

- 手动移除（removeObjectForKey）
- 缓存超过设定的上线
- App不活跃
- 系统内存爆炸

```objectivec
@property BOOL evictsObjectsWithDiscardedContent;
```

该属性默认为True ， 表示在内存销毁时丢弃该对象 。

```objectivec
@property NSUInteger totalCostLimit;
```

总成本数 ， 用来设置最大缓存数量

### 开始使用NSCache

```objectivec
//
//  TDFSetPhoneNumController.m
//  TDFLoginModule
//
//  Created by doubanjiang on 2017/6/5.
//  Copyright © 2017年 doubanjiang. All rights reserved.
//

#import "viewController.h"

@interface viewController () <NSCacheDelegate>
@property (nonatomic ,strong) NSCache *cache;
@end

@implementation viewController

- (void)viewDidLoad {
    
    [super viewDidLoad];
    
    [self beginCache];
}

- (void)beginCache {
    for (int i = 0; i<10; i++) {
        NSString *obj = [NSString stringWithFormat:@"object--%d",i];
        [self.cache setObject:obj forKey:@(i) cost:1];
        NSLog(@"%@ cached",obj);
    }
}

#pragma mark - NSCacheDelegate

- (void)cache:(NSCache *)cache willEvictObject:(id)obj {
    
    //evict : 驱逐
    NSLog(@"%@", [NSString stringWithFormat:@"%@ will be evict",obj]);
}


#pragma mark - Getter

- (NSCache *)cache {
    if (!_cache) {
        _cache = [NSCache new];
        _cache.totalCostLimit = 5;
        _cache.delegate = self;
    }
    return _cache;
}
@end
```

我们会看到以下输出

```objectivec
2018-07-31 09:30:56.485719+0800 Test_Example[52839:214698] object--0 cached
2018-07-31 09:30:56.485904+0800 Test_Example[52839:214698] object--1 cached
2018-07-31 09:30:56.486024+0800 Test_Example[52839:214698] object--2 cached
2018-07-31 09:30:56.486113+0800 Test_Example[52839:214698] object--3 cached
2018-07-31 09:30:56.486254+0800 Test_Example[52839:214698] object--4 cached
2018-07-31 09:30:56.486382+0800 Test_Example[52839:214698] object--0 will be evict
2018-07-31 09:30:56.486480+0800 Test_Example[52839:214698] object--5 cached
2018-07-31 09:30:56.486598+0800 Test_Example[52839:214698] object--1 will be evict
2018-07-31 09:30:56.486681+0800 Test_Example[52839:214698] object--6 cached
2018-07-31 09:30:56.486795+0800 Test_Example[52839:214698] object--2 will be evict
2018-07-31 09:30:56.486888+0800 Test_Example[52839:214698] object--7 cached
2018-07-31 09:30:56.486995+0800 Test_Example[52839:214698] object--3 will be evict
2018-07-31 09:30:56.487190+0800 Test_Example[52839:214698] object--8 cached
2018-07-31 09:30:56.487446+0800 Test_Example[52839:214698] object--4 will be evict
2018-07-31 09:30:56.487604+0800 Test_Example[52839:214698] object--9 cached
```

### 总结

经过实验我们发现NSCache使用上性价比还是比较高的，可以在项目里放心去用，可以提升app的性能。