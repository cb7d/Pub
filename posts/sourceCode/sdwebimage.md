---
title: "SDWebimage 的高明之处"
date: 2019-06-18T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","iOS","ObjC","源码阅读"]
---

# SDWebImage

在上古时代，最粗暴的为imageView添加图片的姿势是这样的

```objectivec
NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:@"https://github.com/FelixScat/Pub/blob/master/image/retainCircle.png?raw=true"]];
[self.imgV setImage:[UIImage imageWithData:data]];

```

而目前大部分iOS应用中都使用了网络图片缓存框架，使用最多的莫过于 **SDWebImage** 了，这篇主要从源码角度分析下，因为源码比较多，下文例子中可能会进行一些删减

## 调用接口

使用 SD 发起图片请求仅仅需要一行代码

```objectivec
[self.imgV sd_setImageWithURL:[NSURL URLWithString:@"https://cn.bing.com/sa/simg/hpc26.png"]];
```

我们可以追踪进文件 **UIImageView+WebCache** 里面

```objectivec
- (void)sd_setImageWithURL:(nullable NSURL *)url {
    [self sd_setImageWithURL:url placeholderImage:nil options:0 progress:nil completed:nil];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:0 progress:nil completed:nil];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:options progress:nil completed:nil];
}

...
```

这里发现这些方法最终都会调用同一个方法，相当于便利方法，为不同的需求提供多种接口，下面最终的方法是这样的

```objectivec
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
                   context:(nullable SDWebImageContext *)context
                  progress:(nullable SDImageLoaderProgressBlock)progressBlock
                 completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_internalSetImageWithURL:url
                    placeholderImage:placeholder
                             options:options
                             context:context
                       setImageBlock:nil
                            progress:progressBlock
                           completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL) {
                               if (completedBlock) {
                                   completedBlock(image, error, cacheType, imageURL);
                               }
                           }];
}
```

这里调用的`sd_internalSetImageWithURL`方法是 `UIView`的分类，作者应该是希望和`UIButton`复用一份代码，所以把核心逻辑作为了`UIView`的分类

查看`UIView+WebCache.h`这个文件

```objectivec
// 通过runtime为对象添加图片URL属性，只读
@property (nonatomic, strong, readonly, nullable) NSURL *sd_imageURL;
// 动态添加图片进度属性
@property (nonatomic, strong, null_resettable) NSProgress *sd_imageProgress;
// 动态添加图片过渡动画
@property (nonatomic, strong, nullable) SDWebImageTransition *sd_imageTransition;
// 动态添等待菊花
@property (nonatomic, strong, nullable) id<SDWebImageIndicator> sd_imageIndicator;

// 设置图片的核心方法
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock;

// 取消当前加载的图片
- (void)sd_cancelCurrentImageLoad;
```

可以看到，逻辑很清晰，加载图片，取消加载，下面我们查看设置图片的核心方法

```objectivec
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock {
		// 使用 copy 防止传入可变对象,
    context = [context copy]; 
                           
		// 获取验证操作的key（后面会用到很多次），如果没有的话就用 NSStringFromClass 的方法用类名生成key
    NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
    if (!validOperationKey) {
        validOperationKey = NSStringFromClass([self class]);
    }
    self.sd_latestOperationKey = validOperationKey;
   
    // 从Operation的Map中取消加载图片的operation
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
		// runtime setURL
    self.sd_imageURL = url;
    
		// 如果设置了延时加载占位图则不立即占位图片
    if (!(options & SDWebImageDelayPlaceholder)) {
      	// 有关主队列和主线程的安全问题，下面会具体说这块
        dispatch_main_async_safe(^{
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:SDImageCacheTypeNone imageURL:url];
        });
    }
    
    if (url) {
        // 重设进度
        NSProgress *imageProgress = objc_getAssociatedObject(self, @selector(sd_imageProgress));
        if (imageProgress) {
            imageProgress.totalUnitCount = 0;
            imageProgress.completedUnitCount = 0;
        }

        // 获取 SDWebImageManager
        SDWebImageManager *manager = context[SDWebImageContextCustomManager];
        if (!manager) {
            manager = [SDWebImageManager sharedManager];
        }
        
      	// 设置下载进度回调
        SDImageLoaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            if (imageProgress) {
                imageProgress.totalUnitCount = expectedSize;
                imageProgress.completedUnitCount = receivedSize;
            }
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
        @weakify(self);
      	// 正式开始下载
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options context:context progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            @strongify(self);
            if (!self) { return; }
            // 如果还没有更新progress，直接标记为完成
            if (imageProgress && finished && !error && imageProgress.totalUnitCount == 0 && imageProgress.completedUnitCount == 0) {
                imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
            }
            
#if SD_UIKIT || SD_MAC
            // check and stop image indicator
            if (finished) {
                [self sd_stopImageIndicator];
            }
#endif
            
          	// 根据传入的option获取配置项
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!self) { return; }
                if (!shouldNotSetImage) {
                    [self sd_setNeedsLayout];
                }
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, data, error, cacheType, finished, url);
                }
            };
            
						// 在主线程主队列回调，让调用方安全使用
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }
            
            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
              	// 如果传入了 SDWebImageAvoidAutoSetImage 就在这时设置 placeholder
                targetImage = placeholder;
                targetData = nil;
            }
            
#if SD_UIKIT || SD_MAC
            // transition 设置时机
            SDWebImageTransition *transition = nil;
            if (finished && (options & SDWebImageForceTransition || cacheType == SDImageCacheTypeNone)) {
                transition = self.sd_imageTransition;
            }
#endif
          	// 在拿到图片后最终设置图片
            dispatch_main_async_safe(^{
#if SD_UIKIT || SD_MAC
                [self sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
#else
                [self sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:cacheType imageURL:imageURL];
#endif
                callCompletedBlockClojure();
            });
        }];
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } else { // 如果没传URL直接报错
#if SD_UIKIT || SD_MAC
        [self sd_stopImageIndicator];
#endif
        dispatch_main_async_safe(^{
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
                completedBlock(nil, nil, error, SDImageCacheTypeNone, YES, url);
            }
        });
    }
}
```

这里有几个很有意思的点

- `sd_cancelImageLoadOperationWithKey` & `sd_setImageLoadOperation`
- `dispatch_main_async_safe`
- `#if SD_UIKIT || SD_MAC`

### ImageLoadOperation

这是贯穿整个SD的一种管理模式，框架会为View生成一个管理operation的Map

```objectivec
// runtime 关联对象
- (SDOperationsDictionary *)sd_operationDictionary {
    @synchronized(self) {
        SDOperationsDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
        if (operations) {
            return operations;
        }
      	// NSMapTable 是一个使用自由度比较高的Map，可以自定义内存使用策略
        operations = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
        objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        return operations;
    }
}

- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key {
    if (key) {
      	// 添加操作的时候先调用撤销
        [self sd_cancelImageLoadOperationWithKey:key];
        if (operation) {
            SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
          	// 线程安全
            @synchronized (self) {
                [operationDictionary setObject:operation forKey:key];
            }
        }
    }
}
```

### dispatch_main_async_safe

我们查看 dispatch_main_async_safe 这个宏定义的源码

```objectivec
#ifndef dispatch_main_async_safe
#define dispatch_main_async_safe(block)\
    if (dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL) == dispatch_queue_get_label(dispatch_get_main_queue())) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
#endif
```

这个宏的用途顾名思义是为了保障在主线程安全执行代码，按照一般的理解，上面if的判断中使用 `[NSThread isMainThread]` 也是可以的啊，为什么一定要使用 `dispatch_queue_get_label` 来判断呢，这是因为有一些UI的操作不但要在主线程，也一定要在主队列上执行，这时就要将任务添加到主队列以提升框架的健壮性。

### #if SD_UIKIT || SD_MAC

我们可以看到好多地方都有类似的定义，我们可以看这些宏具体的定义

```objectivec
#if TARGET_OS_OSX
    #define SD_MAC 1
#else
    #define SD_MAC 0
#endif

// iOS and tvOS are very similar, UIKit exists on both platforms
// Note: watchOS also has UIKit, but it's very limited
#if TARGET_OS_IOS || TARGET_OS_TV
    #define SD_UIKIT 1
#else
    #define SD_UIKIT 0
#endif

#if TARGET_OS_IOS
    #define SD_IOS 1
#else
    #define SD_IOS 0
#endif

#if TARGET_OS_TV
    #define SD_TV 1
#else
    #define SD_TV 0
#endif

#if TARGET_OS_WATCH
    #define SD_WATCH 1
#else
    #define SD_WATCH 0
#endif
```

作者是自定义了一些宏来区分 target，我们开发的时候也可以这样多用一些自己定义的宏，不要直接去使用系统提供的，这样在以后的维护中也会方便一些

## SDWebImageManager

下面让我们回到刚才获取图片的方法

SDWebImageManager.m

```objectivec
- (SDWebImageCombinedOperation *)loadImageWithURL:(nullable NSURL *)url
                                          options:(SDWebImageOptions)options
                                          context:(nullable SDWebImageContext *)context
                                         progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                        completed:(nonnull SDInternalCompletionBlock)completedBlock {
    // Invoking this method without a completedBlock is pointless
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

		// 防止调用方直接传了NSString类型进来
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // 如果生成的 URL 是 NSNull 的话，直接设置url为空
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }
		//操作的对象，看过AFN源码的同学可能会有种似曾相识的感觉
    SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    operation.manager = self;
		
		// 确认是否已经加载并且失败过
    BOOL isFailedUrl = NO;
    if (url) {
      	// 加锁，使用 GCD 信号量
        SD_LOCK(self.failedURLsLock);
        isFailedUrl = [self.failedURLs containsObject:url];
        SD_UNLOCK(self.failedURLsLock);
    }
		
		// 这里如果url地址为空字符串 || （配置项为设定失败后重试 && 已经失败过）就直接返回错误信息
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}] url:url];
        return operation;
    }
		
		// 同样适用GCD信号量加锁，将操作对象加入 MutableSet
    SD_LOCK(self.runningOperationsLock);
    [self.runningOperations addObject:operation];
    SD_UNLOCK(self.runningOperationsLock);
    
		// 这里最终会用所有的选项和上下文对象生成resylt
    SDWebImageOptionsResult *result = [self processedResultForURL:url options:options context:context];
    
		// 从缓存加载，传入进度和完成的回调
    [self callCacheProcessForOperation:operation url:url options:result.options context:result.context progress:progressBlock completed:completedBlock];

    return operation;
}
```

上面就是调用的外部接口的大概雏形了，下面我们深入缓存，看看SD的缓存究竟做了啥

## SDWebImageCombinedOperation

上面我们在 `loadImageWithURL` 中看到初始化了一个 **SDWebImageCombinedOperation** 对象

```objectivec
@interface SDWebImageCombinedOperation : NSObject <SDWebImageOperation>

/**
 Cancel the current operation, including cache and loader process
 */
- (void)cancel;

/**
 The cache operation from the image cache query
 */
@property (strong, nonatomic, nullable, readonly) id<SDWebImageOperation> cacheOperation;

/**
 The loader operation from the image loader (such as download operation)
 */
@property (strong, nonatomic, nullable, readonly) id<SDWebImageOperation> loaderOperation;

@end
```

 这里的两个成员变量，cacheOperation 和 loaderOperation 就是缓存的调用者了，

先来看 cacheOperation 

```objectivec
- (void)callCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                 url:(nonnull NSURL *)url
                             options:(SDWebImageOptions)options
                             context:(nullable SDWebImageContext *)context
                            progress:(nullable SDImageLoaderProgressBlock)progressBlock
                           completed:(nullable SDInternalCompletionBlock)completedBlock {
    // Check whether we should query cache
    BOOL shouldQueryCache = !SD_OPTIONS_CONTAINS(options, SDWebImageFromLoaderOnly);
    if (shouldQueryCache) {
        id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
        NSString *key = [self cacheKeyForURL:url cacheKeyFilter:cacheKeyFilter];
        @weakify(operation);
        operation.cacheOperation = [self.imageCache queryImageForKey:key options:options context:context completion:^(UIImage * _Nullable cachedImage, NSData * _Nullable cachedData, SDImageCacheType cacheType) {
            @strongify(operation);
            if (!operation || operation.isCancelled) {
                // Image combined operation cancelled by user
                [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:nil] url:url];
                [self safelyRemoveOperationFromRunning:operation];
                return;
            }
            // Continue download process
            [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:cachedImage cachedData:cachedData cacheType:cacheType progress:progressBlock completed:completedBlock];
        }];
    } else {
        // Continue download process
        [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
    }
}
```

这里调用了 `queryImageForKey` 这个方法，并把返回值赋值给了 cacheOperation ，这里直接看重点的实现代码

```objectivec
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options context:(nullable SDWebImageContext *)context done:(nullable SDImageCacheQueryCompletionBlock)doneBlock {
    if (!key) {
        if (doneBlock) {
            doneBlock(nil, nil, SDImageCacheTypeNone);
        }
        return nil;
    }
    
    id<SDImageTransformer> transformer = context[SDWebImageContextImageTransformer];
    if (transformer) {
        // grab the transformed disk image if transformer provided
        NSString *transformerKey = [transformer transformerKey];
        key = SDTransformedKeyForKey(key, transformerKey);
    }
    
    // 先从内存缓存中取
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    
    if (image) {
        if (options & SDImageCacheDecodeFirstFrameOnly) {
            // Ensure static image
            Class animatedImageClass = image.class;
            if (image.sd_isAnimated || ([animatedImageClass isSubclassOfClass:[UIImage class]] && [animatedImageClass conformsToProtocol:@protocol(SDAnimatedImage)])) {
#if SD_MAC
                image = [[NSImage alloc] initWithCGImage:image.CGImage scale:image.scale orientation:kCGImagePropertyOrientationUp];
#else
                image = [[UIImage alloc] initWithCGImage:image.CGImage scale:image.scale orientation:image.imageOrientation];
#endif
            }
        } else if (options & SDImageCacheMatchAnimatedImageClass) {
            // Check image class matching
            Class animatedImageClass = image.class;
            Class desiredImageClass = context[SDWebImageContextAnimatedImageClass];
            if (desiredImageClass && ![animatedImageClass isSubclassOfClass:desiredImageClass]) {
                image = nil;
            }
        }
    }
		
  	// 如果已经获取到 image 并且没有设置 SDImageCacheQueryMemoryData直接结束当前方法
  	// 因为只是查询了内存中的内容，并没有用到operation，所以返回的对象为nil
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryMemoryData));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    // 第二步，查询磁盘缓存
    NSOperation *operation = [NSOperation new];
    // Check whether we need to synchronously query disk
    // 1. in-memory cache hit & memoryDataSync
    // 2. in-memory cache miss & diskDataSync
    BOOL shouldQueryDiskSync = ((image && options & SDImageCacheQueryMemoryDataSync) ||
                                (!image && options & SDImageCacheQueryDiskDataSync));
    void(^queryDiskBlock)(void) =  ^{
        if (operation.isCancelled) {
            if (doneBlock) {
                doneBlock(nil, nil, SDImageCacheTypeNone);
            }
            return;
        }
        
      	// 设计大量内存的使用，创建自动释放池有利于内存平稳释放
        @autoreleasepool {
          	// 获取磁盘中的 data
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage;
            SDImageCacheType cacheType = SDImageCacheTypeNone;
            if (image) {
                // the image is from in-memory cache, but need image data
                diskImage = image;
                cacheType = SDImageCacheTypeMemory;
            } else if (diskData) {
                cacheType = SDImageCacheTypeDisk;
                // 解码
                diskImage = [self diskImageForKey:key data:diskData options:options context:context];
              	// 设置中需要在内存中保留一份
                if (diskImage && self.config.shouldCacheImagesInMemory) {
                    NSUInteger cost = diskImage.sd_memoryCost;
                    [self.memoryCache setObject:diskImage forKey:key cost:cost];
                }
            }
            
            if (doneBlock) {
                if (shouldQueryDiskSync) {
                    doneBlock(diskImage, diskData, cacheType);
                } else {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        doneBlock(diskImage, diskData, cacheType);
                    });
                }
            }
        }
    };
    
    // 使用GCD向队列添加任务，根据设置的sync判断同步或异步
    if (shouldQueryDiskSync) {
        dispatch_sync(self.ioQueue, queryDiskBlock);
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}
```

SD 的 image 和缓存处理到这里就差不多，下面我们可以看看处理内存释放的地方，内存释放有两种

- 内存释放
- 磁盘释放

我们可以先看 SDMemoryCache

```objectivec
- (void)commonInit {
    SDImageCacheConfig *config = self.config;
    self.totalCostLimit = config.maxMemoryCost;
    self.countLimit = config.maxMemoryCount;
    
    [config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxMemoryCost)) options:0 context:SDMemoryCacheContext];
    [config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxMemoryCount)) options:0 context:SDMemoryCacheContext];
    
#if SD_UIKIT
    self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
    self.weakCacheLock = dispatch_semaphore_create(1);
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(didReceiveMemoryWarning:)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
#endif
}
```

由于 SDMemoryCache 是继承 NSCache 这个类的，它也具备在内存紧张的时候自动释放内存，有关于 这部分可以看关于[NSCache](https://k.felixplus.top/nscache/) 这里要注意的是 SDMemoryCache 重写了父类的一些方法，相当于扩展了一下 NSCache，增加了一个 weakCache 对象

```objectivec
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    if (key && obj) {
        // Store weak cache
        SD_LOCK(self.weakCacheLock);
        [self.weakCache setObject:obj forKey:key];
        SD_UNLOCK(self.weakCacheLock);
    }
}

- (id)objectForKey:(id)key {
    id obj = [super objectForKey:key];
    if (!self.config.shouldUseWeakMemoryCache) {
        return obj;
    }
    if (key && !obj) {
        // Check weak cache
        SD_LOCK(self.weakCacheLock);
        obj = [self.weakCache objectForKey:key];
        SD_UNLOCK(self.weakCacheLock);
        if (obj) {
            // Sync cache
            NSUInteger cost = 0;
            if ([obj isKindOfClass:[UIImage class]]) {
                cost = [(UIImage *)obj sd_memoryCost];
            }
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}

- (void)removeObjectForKey:(id)key {
    [super removeObjectForKey:key];
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    if (key) {
        // Remove weak cache
        SD_LOCK(self.weakCacheLock);
        [self.weakCache removeObjectForKey:key];
        SD_UNLOCK(self.weakCacheLock);
    }
}
```

关于磁盘缓存，可以回看 SDImageCache ，其中 SDImageCache 监听了应用进入后台的通知，在进入后台时会调用这个方法

```objectivec
- (void)deleteOldFilesWithCompletionBlock:(nullable SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        [self.diskCache removeExpiredData];
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}

#pragma mark - UIApplicationWillTerminateNotification

#if SD_UIKIT || SD_MAC
- (void)applicationWillTerminate:(NSNotification *)notification {
    [self deleteOldFilesWithCompletionBlock:nil];
}
#endif
```

至此，SD的整体流程大致就分析完了，希望能对各位理解网络缓存图片的流程理解提供一点帮助