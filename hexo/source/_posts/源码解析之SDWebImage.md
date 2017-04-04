---
title: 源码解析之SDWebImage
date: 2016-05-26 15:02:53
tags: [SDWebImage]
categories: [源码解析]
---

> Asynchronous image downloader with cache support as a UIImageView category

-------

# & 引子
"一个支持缓存的异步下载图片的UIImageView分类",这是SDWebImage官方的相当保守的自我描述；
看完下图，就可以了解作者有多么的谦虚；
下图我旨在表现SDWebImage的清晰的模块划分和任务分级策略，并不能全面的涵盖它的一切。
![SDWebImage 模块划分与任务分级](/images/14695977227412.jpg)

从标题就可以知晓，如果你只是想了解SDWebImage的用法，那最好直接去看[README.md](https://github.com/rs/SDWebImage/blob/master/README.md);
下面我会按照任务分级和模块划分，从以下几个方面抽丝剥茧：

-------

# & Cache -缓存管理
内存缓存和磁盘都由**SDImageCache**处理，即有关图片缓存的一切，都封装在这个类里了；
## && 一 整体流程

* 缓存命中流程：
    1. 通过外部传入的key，首先检查是否命中内存缓存，命中则直接返回；
    2. 检查是否命中磁盘缓存，未命中则返回；
    3. 命中磁盘缓存，检查是否开启内存缓存，开启则设置内存缓存，并返回命中结果；
    
## && 二 内存缓存
先来说内存缓存，就像[AFN](https://github.com/AFNetworking/AFNetworking)作者所说，[NSCache](http://nshipster.cn/nscache/)就可以处理一切你想要的了；SDImageCache内部的**memCache**是一个**NSCache**对象，因此直接通过**- setObject:forKey:**就可以设置缓存，但是作者使用了**- setObject:forKey:cost:**，用意是通过**const**来向系统传递单一缓存大小，从而通过系统对整个缓存大小的控制，有效控制整个缓存的容量。但是，这就是NSCache最坑的地方，你永远也无法知道，当缓存到达你设置的临界时，到底是哪部分缓存被清除了（清除也有可能不是立即进行的）。瑕不掩瑜吧，看来作者也并没因此放弃对NSCache的使用。另外，提到const，作者提供了一个计算图片大小的方法，很精简，直接贴出来：

```objc
FOUNDATION_STATIC_INLINE NSUInteger SDCacheCostForImage(UIImage *image) {
    return image.size.height * image.size.width * image.scale * image.scale;
}
```
 其实这个方法计算出来的并不是图片所占字节数，而是像素点，SDWebImage官方也在[此处](https://github.com/rs/SDWebImage/issues/1369)进行了说明；所以，如果要计算图片所占的大小，可以这样做：

```objc
// 延续作者的思路
image.size.height * image.size.width * image.scale * image.scale * CGImageGetBitsPerPixel(image.CGImage) / CHAR_BIT
// 更简洁的方式
CGImageGetBytesPerRow(image.CGImage) * CGImageGetHeight(image.CGImage)
```
以上都是题外话，因为这个计算方法只用在了内存缓存，上面也提到了NSCache的坑，所以此处，这个值也就无所谓了。
另外，关于内存缓存删除的触发，除了NSCache的自动处理外（这部分毕竟是不可控的），作者还监听了**UIApplicationDidReceiveMemoryWarningNotification**做了更加保险的处理。
## && 三  磁盘缓存
较之内存缓存成熟的NSCache方案，磁盘缓存要相对复杂一些了；
因为作者旨在实现的是一个图片缓存管理，相对来说，图片属于大容量对象存储。因此，这里作者直接使用文件存储这种方式，而不是像其他缓存策略那样，数据库和文件并用，也是合理的。
1. 存储
    存储部分，剖去外层一系列的判断，最终会调用下面的方法：  

    ```objc
    - (void)storeImageDataToDisk:(NSData *)imageData forKey:(NSString *)key {
        if (!imageData) {
            return;
        }
        if (![_fileManager fileExistsAtPath:_diskCachePath]) {
            [_fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
        }
        // get cache Path for image key
        NSString *cachePathForKey = [self defaultCachePathForKey:key];
        // transform to NSUrl
        NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
        [_fileManager createFileAtPath:cachePathForKey contents:imageData attributes:nil];
        // disable iCloud backup
        if (self.shouldDisableiCloud) {
            [fileURL setResourceValue:[NSNumber numberWithBool:YES] forKey:NSURLIsExcludedFromBackupKey error:nil];
        }
    }
    ```

    贴出这个方法，是为了说明，作者并没有在每次的磁盘缓存中去做缓存容量和时间的判断，而是很粗暴的进行了文件存储。
    为什么会这么做？所有对磁盘缓存的操作，都放在了**ioQueue**这个串行队列中，试想，如果每次存储都做缓存时效和空间大小的判断和处理，那这个存储效率就太低了，同时缓存的维护代价会变得非常大。当然，外部设置的缓存时效和空间限制也并不是形同虚设，稍后会在缓存清理中做解释。
2. 缓存清理
先解释上面的问题，缓存清理会在应用进入后台和应用即将被终止时进行。
因为作者监听了**UIApplicationDidEnterBackgroundNotification** 和 **UIApplicationWillTerminateNotification**这两个通知。
这两个通知，最终都会调用**cleanDiskWithCompletionBlock**方法，这个方法是整个缓存管理类营养价值最高的地方。由于代码量过大，这里就不贴出来了。下面提几个关键点：
    *         缓存清除策略:时间维度(优先考虑过期时间)和空间维度(LRU逐步衰减)；
    *         清理工作不会立即进行，考虑I/O阻塞和数组遍历修改易crash等因素，待清除对象最后统一处理。
    *         采用LRU,而非LFU的原因：文件缓存天然具有修改时间监控优势，这种实现方式维护成本更低。

<!-- more -->

## && 四  几个细节
   * **ImageDataHasPNGPreffix**是一个判断图片使用是PNG;(PNG图片前8个字节是固定的)
   *  缓存的key，用的URL的MD5值；
   *  **SDDispatchQueueSetterSementics** 是个历史原因，因为起初GCD并不在ARC的管理范围

## && 五 总结：
整个缓存管理类，采用单例模式，在内存缓存中通过NSCache，磁盘缓存中借助本地文件，完成了**时间维度**和**空间维度**双重限制下完整的图片缓存实践；此类中作者给出了在**文件操作**通知处理等教程般的编程示例，值得细细品味。

-------

# & Downloader - 下载管理
下载相关的操作封装在类**SDWebImageDownloader**和**SDWebImageDownloaderOperation**中，前者是后者多任务操作的封装管理者。
这部分精彩的是作者对**NSOperationQueue**和**GCD**的玩儿法！
## && 一 SDWebImageDownloader
* 先来看下载任务管理类SDWebImageDownloader，主要讨论三个属性两个方法
    1. downloadQueue
        这是个NSOperationQueue类型的并发队列，默认的最大任务数是6个；下载相关的直接操作是在这个队列里进行。
    2. URLCallbacks    
        这个字典里存放的是每个下载任务的进度回调和完成回调，具体格式如下：
      
        ``` json
        {
            "URL0": "[{"progress":progressBlock, "completed":compeletBlock}, {"progress":progressBlock, "completed":compeletBlock}]", 
            "URL1": "[{"progress":progressBlock, "completed":compeletBlock}, {"progress":progressBlock, "completed":compeletBlock}]"
        }
        ```
        考虑到会有多个下载任务同时对**URLCallbacks**进行操作，通过**dispatch_barrier_sync**设置屏障来保证线程的安全性。
    3. barrierQueue
        **dispatch_queue_t**类型的并发调度队列；图片下载的回调都在这个队列处理。
        
    4. addProgressCallback: completedBlock:forURL: createCallback:
        这个方法主要处理**URLCallbacks**，如果URL对应的数组第一个item进入时，会执行**creatCallback**回调。代码如下：
        
    ```objc
    - (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
       ...
       // first 判断
        dispatch_barrier_sync(self.barrierQueue, ^{
            BOOL first = NO;
            if (!self.URLCallbacks[url]) {
                self.URLCallbacks[url] = [NSMutableArray new];
                first = YES;
            }
        //  URLCallbacks 的设置
            NSMutableArray *callbacksForURL = self.URLCallbacks[url];
            NSMutableDictionary *callbacks = [NSMutableDictionary new];
            if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
            if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
            [callbacksForURL addObject:callbacks];
            self.URLCallbacks[url] = callbacksForURL;
            // 是否调用回调
            if (first) {
                createCallback();
            }
        });
    }
    ```
    5. downloadImageWithURL: options: progress: completed:
这个方法是外部执行下载任务调用的方法，其实它的主要部分就是上面提到的**createCallback**任务。这里只贴出此方法的主要部分，具体可[戳这里](https://github.com/rs/SDWebImage/blob/master/SDWebImage/SDWebImageDownloader.m)。

    ```objc
    - (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock {
     ...
    // 下面的整个部分都是createCallback
        [self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^{
     ...
            // 1.创建请求对象 options的设置为了避免NSURLCache+SDImageCache的重复缓存，因此默认不进行NSURLCacke的缓存
            NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
             ...
            // 2.创建SDWebImageDownloaderOperation对象 相关配置会在下面对此类的分析中提到
            operation = [[wself.operationClass alloc] initWithRequest:request
                                                            inSession:self.session
                                                              options:options
                                                             progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                                 ...
                                                                // 3.回调此URL的所有进度回调
                                                                 for (NSDictionary *callbacks in callbacksForURL) {
                                                                     dispatch_async(dispatch_get_main_queue(), ^{
                                                                         SDWebImageDownloaderProgressBlock callback = callbacks[kProgressCallbackKey];
                                                                         if (callback) callback(receivedSize, expectedSize);
                                                                     }
                                                            completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                                                                ...
                                                               // 4. 任务完成后删除callbacksForURL中该URL
                                                                dispatch_barrier_sync(sself.barrierQueue, ^{
                                                                    callbacksForURL = [sself.URLCallbacks[url] copy];
                                                                    if (finished) {
                                                                        [sself.URLCallbacks removeObjectForKey:url];
                                                                    }
                                                                });
                                                                // 5.回调此URL对应的所有完成回调
                                                                for (NSDictionary *callbacks in callbacksForURL) {
                                                                    SDWebImageDownloaderCompletedBlock callback = callbacks[kCompletedCallbackKey];
                                                                    if (callback) callback(image, data, error, finished);
                                                                }
                                                            }
                                                            cancelled:^{
                                                                 ...
                                                                // 6.取消操作并将URL对应的回调信息删除
                                                                dispatch_barrier_async(sself.barrierQueue, ^{
                                                                    [sself.URLCallbacks removeObjectForKey:url];
                                                                });
                                                            }];
              ...
            // 7. 将此操作加入到下载队列中
            [wself.downloadQueue addOperation:operation];
            if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
               // 8. 如果任务执行顺序是LIFO则将此操作作为原队列的最后一个任务依赖，并将此操作作为最后一个任务
                [wself.lastAddedOperation addDependency:operation];
                wself.lastAddedOperation = operation;
            }
        }];
    
        return operation;
    }
    ```
SDWebImageDownloader的主要逻辑就在这里了，仔细分析不难发现上面的三属性两方法是整个下载管理器的灵魂。这种将回到通过URL标识，暂存然后统一回调的精妙设计，有效的避免了同一个请求的多次下载操作。
    
## && 二 SDWebImageDownloaderOperation
SDWebImageDownloaderOperation是**NSOperation**的子类，主要处理图片下载的单个任务操作，此任务最终会被添加到**SDWebImageDownloader**的下载队列中。 外部暴露的主要方法是SDWebImageDownloader用到的**initWithRequest:inSession:options:progress:completed:cancelled**，而此方法主要做了属性的初始化工作。下载任务的真正逻辑在重写的**NSOperation** 的**start**方法中。
1. start
    start方法主要做了下面几件事：
    (1) 设置应用进入后台的相关持续执行处理；
    (2) 创建 **NSURLSession**（旧版本使用的**NSURLConnetion**），如果外部传入（SDWebImageDownloader），使用外部对象；
    (3) 开启下载任务，并发送任务开启通知；
2. NSURLSessionDataDelegate 和 NSURLSessionTaskDelegate
    **NSURLSession**不再向NSURLCollection那样会有数据加载完成即finishLoading的回调；而是在NSURLSessionTaskDelegate的**URLSession:task:didCompleteWithError:**代理中error的判断和**completionBlock**的回调；**URLSession:dataTask:didReceiveData:**主要做数据回传后的图片转换的解码缓存相关处理，同时回调**progressBlock**；
    
3. 一些细节
     (1) SDWebImageDownloaderOperation是一个可并发操作对象，通过对各个属性的重写，外部可通过**KVO**监听状态变化；
     (2) 对返回的**304**请求响应做了读取本地缓存处理；
     (3) 下载任务从开始到结果回调的每个任务，都抛出了**通知**；
     (4) **thread**属性指向当前任务执行线程，这在任务取消中做判断处理；
     (5) **beginBackgroundTaskWithExpirationHandler**申请更长的后台任务处理时间；

使用NSOperation，在自定义队列中做并发任务时，SDWebImageDownloaderOperation 是 NSOperation子类化的最佳实践。

## && 三 小结
下载管理器，通过单个图片下载操作**SDWebImageDownloaderOperation**和多个下载任务的管理**SDWebImageDownloader**封装，只暴露了简洁的接口，而把复杂留给自己，内部结合**GCD**和**NSOperation**，对任务做到精确管理。其实聊到这里，SDWebImage的精华已经几乎被我们榨干了，后续的上层封装所做的一切都要依靠Cache和Downloader来做；

-------

# & SDWebImageManager
现在开始探讨题图所示的第二级封装，这一层主要有两个单例：**SDWebImageManager** 和 **SDWebImagePrefetcher**。
## && 一 SDWebImageManager
SDWebImageManager在上层分类调用缓存和下载器的起到了传递作用，同时在下载和缓存之间起到了连接作用，旨在把最简洁的接口暴露给外部；
此类针对下载其实只暴露了一个方法：**downloadImageWithURL:options:progress:completed**，这个方法做了下面几件事：
1. 过滤曾经请求失败过的URL，满足条件会直接返回，这样避免了多次的重复无效请求；但是，同时也会因此降低重试的成功率，不过这个特性可以通过option配置；
2. 调用SDImageCache去尝试命中缓存，如果命中先处理completedBlock，然后将下载任务从**runningOperations**任务管理数组中移除；
3. 如果未命中，则处理options相关配置，然后发起下载任务；

上面提到了任务管理数组runningOperations，这个数组主要是为了管理发起的下载任务，控制任务的手动**cancel**等操作。
一些细节：为了方便cache命中任务和下载任务的管理，尤其是取消操作，搞了个**SDWebImageCombinedOperation**，内部只实现了任务的取消，同时在回调中或者代理中去取消下载的任务；
## && 二 SDWebImagePrefetcher
在需要图片预取的场景中，SDWebImagePrefetcher就派上用场了；其实内部就是个SDWebImageManager的封装，因此即便你不知道有作者会如此贴心，也可以直接用SDWebImageManager实现这个简单的功能。不过，有碰到图片预取的场景时，还是建议使用这个方便的单例类，因为它补充了SDWebImageManager没有涉及的多图预下载，同时还对下载优先级和下载并发数进行了管理。
## && 三 小结
这部分是整个框架的枢纽，在这个层级更多需要在框架的设计和模块组织上思考作者的思路，因为细节部分大多都在底层的缓存和下载器中处理好了。其实，作者在这个层级的表现并没有底层的缓存精彩，主要表现在太多状态的设置和各种无用函数的填充使本不该复杂的实现，变得难以维护。SDWebImageCombinedOperation也显得很鸡肋，感觉作者想通过它做更多事情，但是却又都直接暴露在了SDWebImageManager中，这样使得SDWebImageManager过于臃肿！

-------

# & Category
来到框架的第三层，现在估计就是使用此框架的小伙伴们最熟悉的部分了，每个分类依然保持了对外简洁的作风。
这里想谈的并不多，说一个SDWebImage在tableView的cell重用是如何保证异步图片加载不错乱的经典问题吧；
UIImageView最终会调用分类中的sd_setImageWithURL:placeholderImage:options:progress:completed:方法，此方法最关键的代码：

```objc
// 这个方法的第一行；用意是在同一个ImageView开启下一个任务时，先取消上一个任务；
[self sd_cancelCurrentImageLoad];
// 下面是取消操作所做的事情，这个方法在UIView的分类中，原因是UIButton的分类也会用到，所以写在基类中更合适；
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key {
    // 1.获取操作的维护字典
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    // 2.拿到同一个URL所有的任务
    id operations = [operationDictionary objectForKey:key];
    if (operations) {
    // 3.遍历移除
        if ([operations isKindOfClass:[NSArray class]]) {
            for (id <SDWebImageOperation> operation in operations) {
                if (operation) {
                    [operation cancel];
                }
            }
            // 不是数组的情况下，判断是否实现了SDWebImageOperation协议
        } else if ([operations conformsToProtocol:@protocol(SDWebImageOperation)]){
            [(id<SDWebImageOperation>) operations cancel];
        }
        // 4. 将此URL从字段中移除
        [operationDictionary removeObjectForKey:key];
    }
}
```

-------

# & Tools
作者在搭建整个框架的过程中，抽离出来多个工具类，这部分代码，营养价值依然很高，下面提到几个有代表性的；
## && 一 SDWebImageDecoder
此类提供了一个解码**UIImage**的方法，这里牵涉到这样一个细节：**NSData**转换得到的UIImage对象，在用UIImageView等渲染到屏幕上时，中间还有个**解码**获取**位图**的过程。如果这里不做这个解码操作，这个操作就会滞后到主线程去做，这显然会产生阻塞。
**AFN**也有个类似的方法，但是比这里做的好的地方是，超过1M的大图片，就不再走解码流程，因为位图会占用大量的内存，有可能直接造成**OOM**；这也是SDWebImage始终存在的一个问题，加载大图片时内存暴增，不过可喜的是这个解码属性是可配的。
另外，下载器在图片下载完成后，直接调用了缓存；这样造成的结果就是会有频繁的解码操作，提供个思路，可以直接把位图缓存到磁盘。
## && 二 ImageContentType
这是个判断图片格式的方法，思路很简单，图片格式的第一个字节都是固定的。作者在实现主体功能时，不断将工具方法独立出来的做法值得效仿推崇。关于图片类型的判断方法，[戳这里](https://github.com/rs/SDWebImage/blob/master/SDWebImage/NSData%2BImageContentType.m)
## && 三 MultiFormat
这个类是对NSData转换成各种格式的图片数据做了封装，内部还继续抽离出了**GIF**和**WebP**的处理。还有一些处理图片方向**UIImageOrientation**等细节的方法。

-------

# & 最后
**简洁的接口，清晰的模块划分**；已经不是第一次品读SDWebImage了，随着代码量的增加，看问题的角度也在发生变化，所以每次都能看到些不一样的东西。我相信，给足够的时间，有点基础的人是可以完成一个完整的图片下载框架的。那我们该思考的是：**你的实现，可以如这般精妙吗？**




        



